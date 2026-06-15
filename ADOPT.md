# Adopt it in your project

Four steps to wire this protocol into any project, any AI tool.

---

## Step 1 — Put the protocol in your repo

```
your-project/
├─ docs/
│  └─ proof-of-done.md      ← copy this repo's PROTOCOL.md (or link to this repo)
├─ raw/                     ← archive of user-provided source data (see R4)
└─ <AI rules file>          ← the trigger word goes here (Step 3)
```

You can `git submodule` this repo, or copy `PROTOCOL.md` + `IRON-LAWS.md` into `docs/`. Copying is recommended — you'll keep **appending your own project's hard-won laws to IRON-LAWS** (Step 4).

---

## Step 2 — Map the generic steps to your stack

The steps in the protocol are abstract. Fill them in with your team's concrete commands:

| Generic step | Tools you might use (pick any) |
|---------|-------------------|
| **Deploy** | `docker compose up` / `kubectl rollout` / Helm upgrade / Vercel·Netlify deploy / PaaS push |
| **Restart service** | `docker compose restart` / `kubectl delete pod` / `systemctl restart` / platform restart |
| **Reverse-proxy reload** | `nginx -s reload` / Ingress reload / ALB refresh / API Gateway redeploy |
| **Call a function in the env** | `docker exec` / `kubectl exec` / SSH / serverless invoke |
| **Query logs** | `docker compose logs` / `kubectl logs` / Loki / CloudWatch / Datadog |
| **Query the database** | `docker exec psql` / cloud console / DBeaver / TablePlus |
| **Verify email** | recipient confirmation / IMAP fetch / SendGrid·SES·Postmark dashboard |
| **Verify notification** | Slack / Teams / chat receipt or bot confirmation |
| **Independent AI review** | a second AI CLI (Codex / Claude / Gemini / Aider …) |
| **Browser testing** | Playwright / Cypress / Puppeteer / real browser + DevTools |
| **Bypass self-signed cert** | type `thisisunsafe` on Chrome's cert page; or give the internal domain a trusted cert |

Put this filled-in table at the top of your `docs/proof-of-done.md` so the agent knows how each step maps to your environment.

---

## Step 3 — Wire the trigger word into your AI rules file

Where each AI tool reads its rules:

| AI tool | Rules file |
|--------|---------|
| Claude Code | `CLAUDE.md` (repo root) |
| Cursor | `.cursorrules` |
| Cline | `.clinerules` |
| Aider | `.aider.conf.yml` or `CONVENTIONS.md` |
| GitHub Copilot | `.github/copilot-instructions.md` |
| Continue | `.continue/config.json` (system prompt) |
| Codex CLI | `AGENTS.md` or `CODEX.md` |

Paste the contents of [templates/trigger-rule.md](templates/trigger-rule.md) in. The point: **one trigger word → points at `docs/proof-of-done.md` → the agent runs the whole thing**, so you don't manually nag it to "go verify the database" / "go check the logs" every time.

---

## Step 4 — Keep distilling lessons (and sync them the right way)

Every time you catch a new "agent-reports-done-but-wrong" pattern → append a new law. But **where** it goes is what keeps every project consistent instead of forking:

- **Generic** trap (would bite any stack) → append `R{n}` to the **base** `IRON-LAWS.md`, bump `VERSION`, push. Every project picks it up on its next sync.
- **Project-specific** trap (your stack / ERP / data shape) → put it in **your overlay** only.

And **before** each run, pull the base so you're on the latest laws; pin your overlay to the base `VERSION` you synced. The full loop is in **[SYNC.md](SYNC.md)**.

This is the real meaning of "long-term": **it's not a frozen checklist, it's a knowledge base that grows as you hit new traps** — one shared upstream, many project overlays. R1–R24 grew exactly this way.

---

## Decisions you make yourself

| Option | Trade-off |
|------|------|
| Trigger word language | team habit (e.g. "run the gauntlet" / a phrase in your language) |
| Archive reports or not | archiving helps retros but adds work (suggest `reports/<date>-<topic>.md`) |
| Which two AIs for review | depends on what you subscribe to |
| How much automated smoke testing | more coverage saves human time but costs maintenance |
| Test data in git or not | depends on confidentiality + file size (sensitive data → `.gitignore`) |
| Run Phase 6 browser test every time | backend-only changes can skip; UI changes **must** |
| Max rounds for Phase 4 review | by complexity, commonly 3–5 |

---

## How this differs from a "hardcoded" version

This protocol deliberately abstracts away everything environment-specific, so any project can use it:

| A hardcoded version writes | The generic version abstracts to |
|----------|-----------|
| `docker compose restart backend + nginx -s reload` | "deploy / restart / reverse-proxy reload," you fill per stack |
| a specific SMTP + IMAP server address | "delivery side effects require receiver confirmation" |
| one specific AI CLI | "review with a second, independent AI," your pick |
| a specific prod IP / domain | "the prod / staging / demo environment" |
| a specific sign-token function + test account | the "auth strategy" three-tier priority |

> In one line: **you fill in the environment details, the methodology is given.**
