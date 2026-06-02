# Trigger-word rule template

Paste the block below into your AI rules file (`CLAUDE.md` / `.cursorrules` / `.clinerules` / `AGENTS.md` / `.github/copilot-instructions.md` …). Change the trigger word to fit your team.

---

```markdown
## Automated behavior: long-term test (proof-of-done)

When the user says "run the gauntlet" / "long-term test" / "run the full test"
(or your own trigger word), run all 8 phases (Phase 0–7) of
`docs/proof-of-done.md` end to end, without asking again.
Precondition: the change is already implemented.

Iron laws (front and center, obey every time):
- Local compile / unit / type-check is NOT verification — run it against the
  target environment (prod / staging)
- HTTP 200 / status=sent / commit-done ≠ correct result — verify the PRODUCT
- Delivery side effects (email / notification / webhook) require the receiver
  to confirm receipt, otherwise mark "⏳ awaiting confirmation," not "✅"
- Fix one spot → grep for ALL siblings (Phase 3): a module's endpoints are
  often scattered across multiple files
- If the second AI (Phase 4) is unavailable, switch to adversarial self-review
  — do not skip
- Test-time outbound side effects must not disturb real users (mock delivery,
  but don't touch production scheduled jobs)
- Report in three tiers: ✅ verified / ⏳ awaiting confirmation / ❌ failed-or-untested,
  never merged

8 phases: 0 pre-flight → 1 logic → 2 data accuracy → 3 same-pattern sweep →
          4 independent review → 5 prod E2E → 6 human simulation → 7 sign-off
```

> Tip: keep the trigger word in your team's working language. A non-English example: 「启动长期测试」/「跑长期测试」/「走测试流程」.

---

## Where each AI tool reads its rules

| AI tool | Rules file |
|--------|---------|
| Claude Code | `CLAUDE.md` |
| Cursor | `.cursorrules` |
| Cline | `.clinerules` |
| Aider | `.aider.conf.yml` / `CONVENTIONS.md` |
| GitHub Copilot | `.github/copilot-instructions.md` |
| Continue | `.continue/config.json` (system prompt) |
| Codex CLI | `AGENTS.md` / `CODEX.md` |
