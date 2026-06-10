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
**Recovery recipe (config drift)**: when the second AI's CLI fails on a config it didn't write (e.g. a desktop app wrote config values the older CLI can't parse), don't edit the user's config — run the CLI against an isolated home (`CODEX_HOME=/tmp/<x>` with a minimal config + a *symlink* to the real auth file). Review proceeds; user config untouched.

### R15 — Don't disturb real users during testing
**Trap**: outbound side effects triggered by a test call (mass-emailing real customers / suppliers, pushing a real webhook, mutating a real external system) become incidents — a bunch of real people get baffling test messages.
**General rule**:
- Mock / patch out outbound delivery inside the test call (temporarily swap the send function for a no-op), **scoped to this test process only**
- **Don't change the normal logic of production scheduled jobs** — they should keep sending to real users as usual
- Verify "the logic ran correctly + the right flags got set"; verify delivery itself against a controlled test recipient
**Origin**: a full data-sync would email "order change" notices to real suppliers. During testing, monkeypatched the send function to a no-op (this process only) — verified the sync logic without spamming real suppliers, while the daily scheduled sync kept emailing them as normal.

### R16 — Async reload: don't race the verification in the same command
**Trap**: `openresty -s reload` / `nginx -s reload` / many daemon reloads return **before** workers finish swapping config. Chaining `reload && curl <assert>` in one command can hit the *old* config and "fail" (or worse, "pass" on stale state).
**General rule**: reload and verification are **two separate commands**; insert an explicit wait or poll (`sleep` + retry) between them. Any "apply config → assert behavior" pair must assume the apply is asynchronous unless proven otherwise.
**Origin**: an OpenResty route change "didn't take effect" during verification — it had, but the assertion ran in the same compound command before the graceful reload finished.

### R17 — Programmatic uploads / writes must set MIME explicitly
**Trap**: `curl -F` and most scripted multipart clients send `application/octet-stream` unless told otherwise. Frontends that branch on `content_type` then silently treat the file as "unknown type" — the upload "succeeded" but the feature is broken. Browser form uploads auto-set the type, so manual testing never reproduces it.
**General rule**: every scripted upload sets an explicit MIME (`-F "f=@x.png;type=image/png"`); after upload, verify the **stored** content type, not the HTTP 200.
**Origin**: a screenshot uploaded via `curl -F` stored as `octet-stream`; the UI's type-based preview logic showed nothing. Browser uploads worked, masking the bug from manual tests.

### R18 — Assert rendered HTML via DOM parsing, not raw-text grep (attribute case/形态 is framework-owned)
**Trap**: SSR frameworks serialize attributes in whatever case/form their internals use — Next.js App Router emits `hrefLang` (camelCase), React may emit boolean attributes differently, etc. A case-sensitive `grep 'hreflang='` over raw HTML returns nothing and you "conclude" the feature didn't ship — a false negative on a working fix (or you ship a "fix" for a non-bug).
**General rule**: assertions about rendered HTML semantics go through a **DOM parser** (`document.querySelectorAll('link[rel=alternate][hreflang]')` in a headless browser, or an HTML parser lib) — that's also what crawlers parse. Raw-text grep is only for "is this byte-string present," never for "does this semantic attribute exist." If you must grep, `grep -i`.
**Origin**: post-deploy verification of hreflang tags — lowercase grep found zero matches on a page that had all 6 alternates as `hrefLang=`. The DOM query in a real browser returned 6, matching what Googlebot sees.

### R19 — Cross-OS file pushes carry hidden junk: strip metadata sidecars at the source
**Trap**: pushing files from macOS to a Linux server via tar/zip silently includes AppleDouble sidecars (`._foo`, `.DS_Store`). They land in the server's working tree, show up in `git status`, and can get swept into the next commit (or worse, get served).
**General rule**: on macOS set `COPYFILE_DISABLE=1` before tar, or `tar --no-xattrs`, or clean after landing (`find . -name '._*' -delete`) — and **check `git status` on the target after any cross-OS push**, before committing anything.
**Origin**: a tar-over-ssh deploy from a Mac dropped `._app`, `._next.config.ts` etc. into a production repo; caught in `git status` right after the commit (the commit itself was clean only because files were added by explicit path).

### R20 — In batch endpoint assertions, classify failures before believing them
**Trap**: a batch URL sweep (`xargs -P10 curl`) reports failures that aren't the target's fault: (a) **uniform 100% failure** usually means the harness is broken (e.g. BSD `sed` not supporting `\?` left `<loc>` tags on every URL → all curls got invalid input → all "000"); (b) **sporadic failures under parallelism** through a CDN/tunnel are transport timeouts, not app errors.
**General rule**: triage batch failures by shape before reporting: 100% fail → first re-verify the harness on one known-good case by hand; sporadic fail → retry the failures **serially with a longer timeout**; only failures that reproduce on serial retry count as real. Never report first-pass batch numbers as final.
**Origin**: one session hit both shapes back-to-back: 98/98 "000" from an unstripped XML tag (harness bug), then 5/98 "000" under `-P10` that all turned 200 on serial retry (transport). The real result was 98/98 healthy.
