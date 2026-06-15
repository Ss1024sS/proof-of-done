# proof-of-done — The Full Protocol (8 phases)

> Say the trigger word (see [templates/trigger-rule.md](templates/trigger-rule.md)) → the agent runs every phase, no step-by-step babysitting.
> **Core: local doesn't count; "success" ≠ "correct"; no evidence, no "pass"; three tiers, never merged.**

Audience: any AI-assisted project (Claude Code / Cursor / Cline / Aider / Codex / Copilot …). Not tied to any stack or domain.

---

## The four iron laws

| Law | Meaning | Anti-example |
|------|------|------|
| **A. Local doesn't count** | Compile / unit / type-check is sanity, not verification | "340 tests passed" ≠ "e2e verified" |
| **B. Success ≠ correct** | HTTP 200 / `status=sent` / commit-done is *technical* success only | `sent` ≠ the recipient received it |
| **C. No evidence, no "pass"** | Every claim needs independently re-checkable proof (log / DB row / screenshot / receipt) | "should be fine" / "in theory correct" doesn't count |
| **D. Three tiers, never merged** | ✅ verified / ⏳ awaiting confirmation / ❌ failed-or-untested — kept separate | Dressing ⏳ up as ✅ is a deceptive report |

---

## Phase 0 — Pre-flight

**Goal**: confirm the change actually reached the environment you're about to test — you're not testing your local working tree.

- [ ] The production **build** passes — not just the type-check. Run the framework build (`next build` / `vite build` / AOT), which exercises prerender / static-export / tree-shake paths `tsc`·`mypy` never run (see [R25](IRON-LAWS.md))
- [ ] Change synced to the target environment (prod / staging / demo)
- [ ] Service process restarted (per your deploy method: container / process manager / PaaS)
- [ ] **New artifact actually live** — hashed chunk name changed / a code marker present in the running image; a failed build + `--force-recreate` silently re-runs the old image and still serves 200 (see [R25](IRON-LAWS.md))
- [ ] Reverse proxy / load balancer reloaded (if routes or static assets changed)
- [ ] Code freshness: local `HEAD` == the version actually running on target (compare by hash / md5, not by gut feeling)
- [ ] DB schema change applied (migration / ALTER) **and confirmed to have taken effect** — query the actual schema, don't just trust "the script ran without error" (see [R12](IRON-LAWS.md))
- [ ] Caches cleared / invalidated (CDN / Redis / in-process)
- [ ] **Rollback ready** (mandatory before any environment with real users): take a fresh backup *before* mutating data + know the rollback command (old image tag / DB snapshot / previous release)
- [ ] **Deploy must not clobber the target's hotfix**: if you deploy by bypassing git (rsync / scp), first compare "the files you're about to push" against "the target's current version" so you don't overwrite someone's live fix (see [R13](IRON-LAWS.md))

**Anti-pattern**: change the code, immediately run tests → you're testing the *previous* version. Something breaks and you discover there's no backup and no rollback plan.

---

## Phase 1 — Logic (function-level)

**Goal**: call functions directly in the target environment, walk every branch — normal / boundary / illegal.

- [ ] Execute in the target environment (not local)
- [ ] Feed normal inputs, check return values
- [ ] Feed boundary values (empty / 0 / max / min / null)
- [ ] Feed illegal values, verify it throws or rejects
- [ ] Every if/else branch is covered

**How (pick by runtime)**:

| Runtime | Direct-call method |
|------------|-----------|
| Container (Docker/Podman) | `docker exec -i <c> <runtime> -` piping a script |
| Kubernetes | `kubectl exec -i <pod> -- <runtime> -` |
| Bare process on a remote host | `ssh host "cd /app && <runtime> ..."` |
| Serverless / FaaS | invoke command / console / SDK after deploy |

---

## Phase 2 — Data accuracy

**Goal**: reconcile key numbers against an **independent source** — verify "is it correct," not "did it return something."

Independent sources: hand calculation / original Excel·CSV·business reference data / a second query path (raw SQL vs API vs reporting system) / upstream system records (ERP·CRM·finance) / a historical baseline (last verified figures).

- **Anti-pattern**: "the API returned 100 rows" — but you don't know if those 100 are right.
- **Right pattern**: "of the 100 rows, I spot-checked 5 against the source documents — every field matches," or "computed net shortage 4000 = SO-open 47072 − stock 43072, matches the hand calc."

---

## Phase 3 — Same-pattern sweep ⭐ (stop "treating only the symptom")

**Goal**: after fixing the spot that errored, actively hunt for "the same problem written differently / located elsewhere" — otherwise you only treat the symptom.

- [ ] Name the **pattern** of this change: e.g. "auth decorator A→B" / "added field X" / "changed query semantics" / "changed UI entry point"
- [ ] **`grep` the whole codebase for that pattern**, judge each hit for whether it needs the same fix — **a module's endpoints are often scattered across multiple files**
- [ ] Cross-file / cross-module / front-back parity: changed the backend, did you miss the frontend? Added a field — did all row types / all exports / all views get it? Changed entry A — what about entry B?
- [ ] Write "the sibling cases found + the verdict (fix / no-fix + why)" into the report — **never default to 'only the spot that errored'**
- [ ] Phase 4's independent review is the cross-check for this step — it does **not** replace doing the grep yourself first

**Why it's its own phase**: agents repeatedly commit the "fix one, miss five in the same module" failure. Example: an auth decorator was changed only in `module.py`, missing a dozen endpoints scattered across `module_extra.py` / `module_derived.py` — the user enabled the module but still couldn't reach the core features.

---

## Phase 4 — Independent review (multi-round)

**Goal**: have an **independent perspective** review the diff and catch what the primary agent missed.

**Why independent**: an AI reviewing its own code has blind spots. A different one (different model / vendor) catches the other's habitual omissions.

**Combos** (pick any two): primary AI + a second AI CLI (Codex / Claude / Gemini / DeepSeek …) / primary AI + a remote LLM.

**Cadence**:
1. Send the diff (`git diff <base>..HEAD`) to the second AI
2. Small prompts focused on one class of issue at a time (SQL injection first, then concurrency, then null-deref) — a huge diff dumped at once loses focus
3. Each round: take the P0/P1/P2 list → fix → re-diff for re-review → only move to the next class once this one is LGTM

**Tooling-compat traps** (been burned): non-ASCII paths / paths with spaces can hang some CLIs → rsync to an ASCII temp path before running.

**When the second AI is unavailable** (dropped connection / rate limit / no subscription): **do not skip this phase**. Switch to **adversarial self-review** — put on the reviewer's hat, `grep` every change-related touchpoint (every `X↔Y` join / every query on the affected field), and interrogate each: "could this miss / cross-contaminate / get the boundary wrong?" See [R14](IRON-LAWS.md).

---

## Phase 5 — Production E2E

**Goal**: hit the target environment's endpoints for real, and **verify side effects to the *result*** — not to the "call returned."

### 5.1 Auth strategy (test-account passwords get rotated)

1. If the server exposes a "sign token" capability → sign a short-lived token in code (most robust, skips login)
2. A dedicated test account + password (locked down, rotated in sync)
3. Reusing a real account (audit / isolation risk, not recommended)

### 5.2 Side-effect verification checklist (by type)

| Side effect | Call-layer evidence (**not enough**) | Result-layer evidence (**required**) |
|----------|------------------|----------------|
| **DB write** | INSERT returns rowcount=1 | SELECT it back, field value matches expectation |
| **Email** | SMTP `status=sent` / 200 | **Recipient confirms it arrived (incl. spam folder)** |
| **SMS / Slack / chat** | API success | Receiver confirms arrival + content correct |
| **File generation** | File path exists | Download, open, verify content / format |
| **Job trigger** | Job ID returned | Job finishes and the output is correct (not just "enqueued") |
| **External system** (CRM/ERP) | Call returns 200 | Log into the external system and see the change with your eyes |
| **Cache** | Write OK | A different process / node can read it |
| **Webhook** | 200 | Target system received it + processed correctly |

### 5.3 Email (the most common faceplant — emphasized separately)

**Iron law**: SMTP accepted ≠ recipient received.

1. Confirm in the backend logs that it actually "went out" (note: mail sent by a subprocess / background worker may not land in the main process log — check the right worker's log)
2. **Have the recipient confirm**, or run an automated smoke test (send to a programmatically accessible mailbox, confirm via IMAP)
3. Until the recipient confirms: mark the email item `⏳ awaiting confirmation` — **never `✅`, never "e2e passed"**

### 5.4 Don't disturb real users

Side effects triggered during testing **must not reach real users** (mass-sending test notifications to real suppliers / customers = an incident). Mock / disable outbound delivery inside the test call (e.g. temporarily patch the send function), but **don't touch the normal logic of production scheduled jobs**. See [R15](IRON-LAWS.md).

---

## Phase 6 — Human simulation (browser / client E2E)

**Goal**: a real user client walks the full flow — verify interaction / visuals / data consistency.

- [ ] Use a **real browser / client** (not a headless HTML parser)
- [ ] Walk the complete business flow (login → main action → verify the artifact → logout)
- [ ] **0 console errors**; network panel free of unexpected 4xx/5xx
- [ ] Data matches the backend (no timezone / format divergence between front and back)
- [ ] Usability (button placement / loading states / friendly error messages)
- [ ] **Real clicks / real typing**, not synthetic events (`dispatchEvent` masks real bugs — see [R10](IRON-LAWS.md))

**Self-signed cert / internal-env tricks**: on Chrome/Edge's cert error page type `thisisunsafe` to bypass; sign a token server-side → `localStorage.setItem` in the browser → jump straight to an internal page (skip login every time).

**Tools**: automated — Playwright / Puppeteer / Selenium; semi-auto — real browser + DevTools. **Keep screenshots as evidence, tied to the report.**

---

## Phase 7 — Sign-off

**Goal**: read-only re-check that every "pass" has real evidence; emit the final three-tier report.

- [ ] Every ✅ has concrete evidence (log line / screenshot / DB query result / receiver confirmation)
- [ ] Nothing "the code looks right" got dressed up as "verified"
- [ ] Nothing "awaiting user confirmation" got merged into "verified"
- [ ] Known issues (unfinished / known bugs / scenarios not run) listed separately, not hidden

Report template: [templates/report-template.md](templates/report-template.md). **Mandatory three tiers**:

| Mark | Trigger | Evidence required |
|------|---------|------------------|
| ✅ **Verified** | Objective evidence, independently re-checkable | log / DB query / screenshot / receiver confirmation |
| ⏳ **Awaiting confirmation** | Subjective judgment or external delivery | pending receiver / reviewer feedback |
| ❌ **Failed / untested** | Failure / exception / not covered | state it plainly |

**Forbidden**: promoting ⏳ to ✅ (reporting "pass" without real proof); disguising ❌ as ⏳ (failure painted as "awaiting confirmation"); hiding ❌ (not mentioning it in the report).

---

## In one line

> **Local pass doesn't count, a successful call doesn't count, no confirmation means it's not "done." Three tiers, never merged. Wire the trigger word into your rules file, let the agent run the whole thing, and you just review the final report.**
