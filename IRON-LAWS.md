# Iron Laws (hard-won from real failures)

> Each one is scar tissue from a real faceplant — an agent reported "done" and was wrong.
> Reading these is more convincing than reading the protocol: you'll see exactly what a "fake pass" looks like.
> **The protocol is *long-term*: every time you catch a new "agent-reports-done-but-wrong" pattern, append a new law.**

---

### R1 — "Local pass" is not "verified"
**Trap**: local uses mocks / fixtures / in-memory DB, never touching the real data source (VPN / real DB / external API). All green locally, can't connect / wrong data in prod.
**General rule**: any "tests pass" must be based on production or a production-equivalent environment.
**Exception**: a pure computation function with no external dependency can be local-only — but the report must say "local sanity, pending prod verification."

### R2 — A successful call ≠ a correct result
**Trap**: technical success (200 / commit / sent) and business correctness (right field / received / accurate calc) are two different things.
**General rule**: verify the **result** of a side effect, not the **call**.

### R3 — Delivery side effects require receiver confirmation
**Trap**: SMTP / webhook / message-queue *accepting* ≠ the receiver *processing successfully*.
**General rule**: the verification endpoint of any "delivery" action is at the receiver, not the sender.
**Origin**: a chase-reminder email — reported "3 emails e2e passed" off `status=sent` alone, recipient got zero. `sent` only means the SMTP server accepted it.

### R4 — Persist user-provided data immediately
**Trap**: a temp dir (`/tmp`) is gone on restart; next time you want it, it's lost.
**General rule**: establish a `raw/source_YYYYMMDD/` convention; archive every file the user hands you (screenshots / Excel / samples) on the spot. Later tests reconcile against these "real samples" — this archive is what makes that possible.

### R5 — Test-account credentials expire
**Trap**: a prod account gets rotated by the business side; a password hardcoded in test code 401s sooner or later.
**General rule**: prefer signing a token in server code (skip login) → then a dedicated test account synced regularly → fallback: when it fails, ask the user, don't retry blindly.

### R6 — Tool / path compatibility
**Trap**: some CLI tools are path-sensitive (non-ASCII / spaces / special chars) and hang or mis-parse.
**General rule**: keep the working path ASCII (`/opt/<proj>`, not a path with CJK chars); rsync to `/tmp/<proj>` before sensitive operations; wrap command paths in quotes.

### R7 — Cross-process logs can be missed
**Trap**: a subprocess / background worker's logs may not flow into the main process log stream. You grep the main log, see nothing, assume it didn't happen.
**General rule**: before trusting a log check, confirm your log collection actually covers the process you're inspecting (async task / subprocess / a different worker).

### R8 — Don't let deploys collide with scheduled jobs
**Trap**: a cron triggered mid-deploy can get `SIGTERM`'d, so a sync / settlement silently doesn't run and nobody notices.
**General rule**: in high-frequency deploy windows give scheduled jobs a `misfire_grace_time` + startup catch-up; or avoid the cron peak.

### R9 — Commit + push by default after a change
**Trap**: when you deploy by bypassing git (rsync etc.), the code drifts ahead of git, and the next session can't tell the real state from git.
**General rule**: at a natural stopping point, commit + push by default — don't wait to be asked. No remote? At least commit locally so `HEAD` == prod; if you can't push, say so plainly — don't pretend you did.

### R10 — Synthetic events ≠ real interaction (UI testing)
**Trap**: you drive a form with `dispatchEvent(new Event('input'))` / JS `.click()` and "it passes," but the real bug is that some browsers fire only `change` (not `input`) on `<select>` (old WebKit / embedded webviews). Synthetic events also mask "did the real click even hit" (handlers on dynamically generated elements may not fire).
**General rule**: UI verification must use **real clicks / real typing** (Playwright/Puppeteer click/fill, not dispatchEvent); bind form events to **both `input` and `change`**; account for embedded browsers (in-app webviews) and old mobile webviews.
**Origin**: a "select material does nothing" form bug — the listener was bound to `input` only, so picking the dropdown did nothing in WebKit-class browsers.

### R11 — Bulk migration / data rewrite safety
**Trap**: running a migration script over existing data (generate aliases / backfill fields / reformat) is high-risk; agents easily "look like they finished" while actually polluting or fabricating data.
**General rule (five checks for migration scripts)**:
1. **Idempotent**: re-run = no-op (skip already-processed), not "one more batch each run"
2. **Spot-check against the source**: reconcile generated values against the original fields independently — "is it correct," not "did it generate something"
3. **Don't pollute non-target rows**: touch only what should change; query "how many out-of-scope rows got changed (expect 0)"
4. **Dry-run must be truly side-effect-free**: don't call state-mutating functions in dry-run (e.g. logic that advances a sequence counter); use a preview / read-only path
5. **Derive fields from the authoritative source**: don't infer classification from dirty data (e.g. a code prefix); use the DB's authoritative field
**Origin**: a few-thousand-row legacy alias migration — the dry-run once advanced the sequence counter; an array got fabricated into a field value; a code-prefix≠actual-category mistake got cemented in.

### R12 — Schema / index changes: "verify it actually took effect," and don't base idempotency on a cached snapshot
**Trap**: a DDL migration (add column / change index / ALTER) running without error ≠ it took effect. Especially "DROP then CREATE" migrations: if you check "does the index already exist" via an ORM's `inspector` cache, that cache may be a **snapshot from before the DROP** — after dropping, the cache still shows the index present, so CREATE gets skipped, leaving **the old constraint dropped and the new one never built**. Uniqueness / the index silently vanishes until prod collides on a key.
**General rule**:
- After a migration, **query the actual schema** (`information_schema` / `pg_indexes` / `\d`) to confirm the structure really changed — don't just trust "the script didn't throw"
- Make idempotency checks **live queries** (does the current index definition contain the new column?), not a possibly-cached metadata snapshot
- Put "DROP + CREATE" behind an idempotent condition (act only when the current state doesn't match the target) to avoid rebuilding on every startup
**Origin**: a unique index was upgraded from 3 columns to 4. The migration used the inspector cache to "skip if the index exists"; after dropping the old 3-column index, the cache still matched → the 4-column index was never built → the unique constraint disappeared. It was caught by querying `pg_indexes` during deploy — fixed by hand-rebuilding it and switching to a "does `indexdef` contain the new column" live-query idempotency check.

### R13 — Before deploying (bypassing git), make sure you don't clobber the target's hotfix
**Trap**: when prod is deployed by rsync / scp (not `git pull`), a blind whole-directory sync can overwrite "a live hotfix someone applied on the target / a change your git doesn't have."
**General rule**:
- Sync the **exact list of files** you changed this round (`rsync --files-from`), not a whole directory with `--delete`
- Before syncing, **compare** "the files you're about to push" against "the target's current version": hash (md5) the target file and compare against your *pre-change baseline* — if they match, the target has no out-of-git changes and overwriting is safe; if not, pull the target's version back and merge first
**Origin**: prod wasn't a git repo (rsync deploy). Before overwriting, md5-compared "prod version vs local pre-commit baseline" for the sensitive files; all matched, so the overwrite was safe.

### R14 — When the second AI is unavailable, substitute "adversarial self-review" — don't skip
**Trap**: independent review (Phase 4) depends on a second AI, but it can drop / rate-limit / be unsubscribed. Agents readily "skip the review" and report done.
**General rule**: when you can't get a result from the second AI, **do not skip** — put on the reviewer's hat and do adversarial self-review:
- List the change's **related patterns** (e.g. "every `X`-to-`Y` join," "every filter on field Z")
- `grep` out every occurrence and interrogate each: "could this miss / cross-contaminate / get the boundary wrong?"
- Focus on "after I changed dimension A, did *every* place that joins on the old dimension keep up?"
**Origin**: the independent-review AI dropped its connection mid-run with no structured result. Switched to hand-grepping every "demand↔order" touchpoint for a deep review, which caught 2 missing isolation conditions (receipt deduction, stale-data cleanup) — both real cross-dimension data-bleed bugs.

### R15 — Don't disturb real users during testing
**Trap**: outbound side effects triggered by a test call (mass-emailing real customers / suppliers, pushing a real webhook, mutating a real external system) become incidents — a bunch of real people get baffling test messages.
**General rule**:
- Mock / patch out outbound delivery inside the test call (temporarily swap the send function for a no-op), **scoped to this test process only**
- **Don't change the normal logic of production scheduled jobs** — they should keep sending to real users as usual
- Verify "the logic ran correctly + the right flags got set"; verify delivery itself against a controlled test recipient
**Origin**: a full data-sync would email "order change" notices to real suppliers. During testing, monkeypatched the send function to a no-op (this process only) — verified the sync logic without spamming real suppliers, while the daily scheduled sync kept emailing them as normal.
