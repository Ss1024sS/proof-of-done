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
**Recovery recipe (config drift)**: when the second AI's CLI fails parsing its own config, first read the error — often the offending key holds a value that was *never* valid locally (e.g. `service_tier = "priority"`: a plan/enterprise-side scheduling entitlement, not a config value; written by a desktop app). The real fix is deleting that key (confirm with the user when it's their config). If you must not touch the user's config mid-task, bridge with an isolated home (`CODEX_HOME=/tmp/<x>` with a minimal config + a *symlink* to the real auth file) — but treat that as a bridge, not the fix; the broken config will bite the next session.

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

### R21 — Framework-injected defaults are sentinel objects in direct-call unit tests
**Trap**: tests that call a route/handler function **directly** (not through the framework) rely on its declared defaults. But a parameter declared `x = Query(None)` / `x = Depends(...)` / `x = Field(None)` defaults to the *framework's sentinel object*, not the resolved value — and that object is **truthy**. A caller that omits a newly-added such param leaves `x` = the sentinel; an `if x:` guard then fires unconditionally (filters apply, branches taken) and breaks **sibling tests that were green a minute ago**, with an opaque type error deep in the stack.
**General rule**: when you add a defaulted framework-injected param, either (a) audit every direct-call test and pass it explicitly, or (b) normalize inside the function (`if x is None or isinstance(x, FieldInfo): x = real_default`). Through the real framework the default resolves correctly — so the breakage shows up *only* in direct-call tests, which is exactly where it's easy to misattribute.
**Origin**: a list endpoint gained two optional `Query(None)` date-filter params. Existing tests called the function directly without them; the params stayed as `Query` objects (truthy) → the date filter ran with a `Query` object as the bound → every sibling rollup test threw "SQLite Date type only accepts Python date objects." Fixed by passing the new params explicitly in the test helper.

### R22 — Fix the bug *class*, and give every sibling an explicit disposition
**Trap**: you fix the reported instance and stop. The same interaction pattern lives in N other places (the same back-button navigation, the same missing filter, the same unguarded fetch); they stay broken — and worse, the report implies "fixed" without naming them. The reviewer/user can't tell what was left untouched.
**General rule**: Phase 3 isn't just "grep" — it's "grep, then **dispose of each hit on the record**": fix now / defer-with-reason / spawn a follow-up task. A sibling you consciously decided not to fix is a *documented decision*; a sibling you never looked for is a *latent bug you'll be asked about later*. Never let "I fixed the one they reported" read as "I fixed the class."
**Origin**: fixing "detail page back-button loses the list's search context" on one screen, a sweep found the same pattern on four other list/detail pairs. They were lower-severity (sensible default landing, not user-reported) → deliberately deferred to a tracked task and written into the report, rather than silently left or hastily batch-changed on user-facing pages mid-test.

### R23 — A new async fetch beside guarded ones must copy the guard
**Trap**: a component already race-guards its fetches (a generation counter / `AbortController`), but a newly-added fetch (a search, a tab-load) skips it. Under fast user input two requests overlap; the slower-older one returns last, **clobbers the newer result** — and if it also writes shared state (URL, cache, selection), it reverts the user's latest action.
**General rule**: when you add an async fetch next to existing ones, replicate the existing concurrency guard — bump a per-fetch generation and drop the response (don't write results / state / URL) if a newer fetch has started. "It usually returns in order" is not a guarantee.
**Origin**: a search box got a new `fetchSearch` while the sibling board-fetch already had a generation ref. Rapid "search → clear filter → search again" could let the first response land last, overwrite the rows, and re-write the *old* filters back into the URL — undoing the user's "show everything." An independent review (Phase 4) caught it; fixed by giving the search its own generation guard.

### R24 — A return/redirect target read from a URL param must be same-origin-validated
**Trap**: to send the user "back where they came from," you read a `?back=` / `?return=` / `?next=` param and navigate to it. Taken raw, a crafted link (`?back=https://evil.example` or `?back=javascript:…`) turns your "back" button into an **open redirect** — phishing-ready, and on an authenticated internal page no less.
**General rule**: never feed a param-sourced URL straight into navigation. Allow only a same-origin internal path (`value.startsWith('/expected/prefix')`), reject anything carrying a scheme or a `//host`, and fall back to a fixed safe route otherwise.
**Origin**: a detail page returned to its list via `router.push(back)` where `back` came from the query string; the first review round had it use `history.length` (unreliable for deep-links), the fix switched to an explicit `?back=` param, and the *second* review round flagged that param as an unvalidated open-redirect. Fixed by gating on `back.startsWith('/<list-route>')` with a safe fallback.

### R25 — A green type-check is not a green build
**Trap**: `tsc --noEmit` / `mypy` / the type-check passes, so you deploy — but the actual production **build** (`next build`, `vite build`, the bundler, an AOT compile) does *more*: static prerendering / route export, tree-shaking, env validation, server-component boundary checks. It fails on things the type-checker never runs. Canonical case: a Next.js App Router page calling `useSearchParams()` **without a `<Suspense>` boundary** — `tsc` is perfectly happy, `next build` throws during static export of that route. And the failure mode compounds: a failed image build followed by `up -d --force-recreate` silently **re-runs the last good image**, so the container is healthy and serving 200s while your new code never shipped.
**General rule**: the gate for deployment is the **framework build**, not the type-checker. Run the real `build` command (locally or in CI) before pushing; treat type-check as a fast pre-filter, not the gate. Build-time-only paths (static generation, prerender) are skipped by both `tsc` and `dev` mode — the only thing that exercises them is the actual build. After deploy, confirm the *new artifact* is live (hashed chunk name changed / a code marker is present in the running image), not just that the service returns 200 — a stale-image rollback looks healthy.
**Origin**: a portal page gained `useSearchParams()` for filter persistence; `tsc --noEmit` passed clean, but the Docker `next build` failed prerendering two routes (App Router requires `useSearchParams` under `Suspense`). The failed build left the previous image running — the smoke test was green, but a grep for the new code marker in the container came back empty, which is what revealed it hadn't shipped. Fixed by wrapping the page in `<Suspense>`, verifying with a local `next build` (all static pages generated), then redeploying and re-checking the chunk hash changed.

### R26 — Ordering by a string-backed enum sorts alphabetically, and LIMIT then truncates the rows you wanted
**Trap**: a status/state column is an enum stored as a **string** (`native_enum=False`, a VARCHAR/CHAR column, a `CharField(choices=...)`, a TEXT-backed enum) — not a DB-native enum or an int. `ORDER BY status` then sorts by the **string value alphabetically**, *not* by declaration order or workflow order. A comment like `# NEW/REVIEWING/.../CLOSED` documents the intended order, but the database does the opposite when the alphabet disagrees (e.g. `CLOSED` < `FULFILLING` < `NEW` — "closed" sorts *first*). On its own this is cosmetic. Combined with **`LIMIT`** it becomes a silent data bug: the rows you actually care about (the active/open ones) sort past the limit window and are never returned. The UI shows an empty "in-progress" list and everyone concludes "no data / page isn't built yet" — when the records exist and are simply truncated.
**General rule**: never `ORDER BY` a string-backed enum and expect lifecycle/priority order. Sort by an explicit expression — `CASE WHEN status = <terminal> THEN 1 ELSE 0 END`, an integer rank column, or a mapped sort key. For any `ORDER BY` + `LIMIT` pair, ask "could the rows I need sort *outside* the limit window?" — if the sort key isn't the dimension you're prioritizing, they can. When a list reads "empty / no data", suspect sort-plus-truncation before concluding the data is missing. Grep every `order_by(<that enum column>)` — the bug usually repeats across the read path and the admin path.
**Origin**: a supplier portal's "in-progress POs" tab showed empty; the IT director reported the page as "still being built, no data". The PO status enum was `native_enum=False` (VARCHAR); `order_by(status)` sorted `CLOSED` first alphabetically, and the default `limit=100` filled entirely with recently-closed POs — the supplier's 152 active POs sorted past row 100 and were never sent. Container-direct-calling the real endpoint against production data (not a local fixture) is what exposed it: 152 active in the DB, 0 in the response. Fixed with `CASE WHEN status==CLOSED THEN 1 ELSE 0 END` active-first ordering across both the portal and admin list paths, plus raising the limit to cover the largest supplier.

### R27 — An exception branch a shared backstop re-blocks is invisible until you construct the input that triggers it
**Trap**: you add a deliberate *relaxation* in one layer — "in this narrow case, allow X that the normal rule forbids" (borrow an out-of-window row to fill a pack, accept a past-dated entry during migration, let an over-limit value through for a VIP). The function implementing the exception is correct in isolation and its unit test is green. But everything funnels through a **shared downstream validator** that re-applies the *base* predicate the exception was meant to bypass — so the operation that should now succeed still gets rejected. The exception is silently nullified by the backstop, and the bug lives only in the *composition*, never in either function alone. Worse, it's **latent**: the exception's precondition is narrow, real/happy-path data never satisfies it, so production never executes the branch — and your Phase-2 data-accuracy scan reads **"0 cases triggered,"** which feels like confirmation when it actually means *the path is entirely untested*.
**General rule**: trace every new exception/relaxation **all the way to its side effect** (the DB write, the HTTP response), not just to the end of the function that implements it — any shared validator the call re-enters will re-run the original rule and undo you. Give the backstop an explicit carve-out (pass the exempted ids/keys down; keep the strict path strict). And treat **"0 occurrences" in an accuracy probe as "unverified," not "fine"**: when current data doesn't exercise a new branch, *make* it — drive the **real deployed code** with a constructed boundary input (shift the date the gate compares against, amplify the parameter, pass a synthetic row/id) and assert the end-to-end outcome, not the isolated function's return.
**Origin**: a supplier "merge-and-ship" flow gained a rule — when in-window shortage is less than one pack, borrow the nearest out-of-window demand rows to round up to a full pack. The borrow function worked and unit-tested clean, but submission reused a shared `_validate_ship_lines` whose window gate re-ran `in_window()` over **every** line; the borrowed out-of-window rows failed it → 400, so the feature never actually shipped a pack. Production hadn't broken because real shortages were already pack-aligned — a full-table scan found **0 of 1789** supplier-item pairs triggering the borrow, which is exactly why it slipped through review and stayed latent. An independent review (Phase 4) flagged the composition; verified by forcing the branch on the live function (shifting the "today" the gate compares against to make real rows fall out-of-window) and confirming the gate rejected without the carve-out and passed with it. Fixed by threading a `window_exception_ids` set from the borrow site through to the backstop, leaving the per-line direct path strict.

### R28 — A format-based cleanup in an idempotent startup migration silently becomes a data-wipe when the canonical format changes
**Trap**: an init/boot migration (the thing that runs on every container start to make the schema/data consistent) contains a "delete rows that don't match the expected shape" statement — `DELETE WHERE length(code) != 2`, `WHERE status NOT IN (...)`, `WHERE version < N`, `WHERE type = 'legacy'`. It encodes the *current* canonical format as a hardcoded constant. Later you migrate that format (2-digit → 3-digit codes, a new enum value, a new id scheme) and update the seed/import to write the **new** shape — but nobody re-reads the boot migration, so its now-stale guard **deletes every newly-written valid row on the next start**. The lethal part is the timing: right after you manually seed/import, the rows are present and everything works — demo passes, manual check passes, tests pass (they run against the just-seeded state). Then the next deploy/restart runs the boot migration and silently wipes it back to defaults. The symptom is "it worked yesterday / it never took in prod despite the seed having run" — intermittent-looking, easy to misattribute to a flaky import.
**General rule**: when you migrate a canonical format (code length, enum set, id scheme, a "legacy vs new" flag), **grep every boot/idempotent migration for format/length/value-predicated `DELETE` and `UPDATE`** — each one hardcodes the OLD format and will fight your new writes on every boot. Flip the guard to the new format or drop it. And make **"survives a restart" an explicit test**: seed/import the data, then *restart the service*, then confirm the rows are still there. One-shot checks against fresh state can't catch a wipe that only fires on the *next* boot — only a restart-then-verify does. (Corollary of R12: "verify the schema change took effect" — here, verify it *stays* in effect after a reboot.)
**Origin**: an SRM "category advance-day" config migrated from 2-digit to 3-digit category codes (matching now keyed on item-code prefix[:3]). The boot migration still ran `DELETE FROM srm_category_advance_windows WHERE length(category_code) != 2`, so every backend restart deleted all the correct 3-digit rows. The seed/import created them (e.g. SMT parts = 90 days), they worked, then the next restart reverted everything to the 10-day default — so prod "never had SMT at 90 days" even though the seed had been run. Caught when a re-import after a restart returned `created=138` instead of `updated=138`, proving the rows had been deleted between runs; fixed by flipping the guard to `!= 3` (purge the now-legacy 2-digit, keep 3-digit) and verified by restarting and confirming all 138 rows + the 90-day values survived.
