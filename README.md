# proof-of-done

> **Local pass ≠ verified. HTTP 200 ≠ correct result. No receipt ≠ "done".**
> A tool-agnostic, domain-agnostic verification protocol for AI-assisted development — so that "done" means *provably done*, not "the code compiles and the tests are green."

[中文文档 →](#中文文档) · [The Protocol](PROTOCOL.md) · [Iron Laws](IRON-LAWS.md) · [Adopt it](ADOPT.md)

---

## The problem this solves

The #1 failure mode of AI coding agents: treating **"code written + compiles + unit tests pass"** as **"task complete."** It almost never is.

| The agent reports "done" | What's actually true |
|---|---|
| Local test suite is green | Can't reach the real DB in prod / the data is wrong |
| HTTP endpoint returns `200` | Business result is wrong (bad field, off-by calc) |
| Email API says `status=sent` | The recipient's inbox got nothing |
| `INSERT` committed | The persisted value doesn't match expectation |
| File downloaded successfully | The content is corrupt or empty |
| Frontend builds clean | Browser throws red errors / data never renders |

**proof-of-done** writes these traps into a hard, repeatable process. The agent can't shortcut it, and the "pass" you get back is a *real* pass — backed by independently verifiable evidence.

## How it works in one line

Register a **trigger word** in your AI rules file → say it once after a change → the agent runs all **8 phases** → you only review the final **three-tier report**.

```
You: "run the gauntlet"
Agent: [Phase 0 deploy → 1 logic → 2 data → 3 same-pattern sweep →
        4 independent review → 5 prod E2E → 6 real browser → 7 sign-off]
Agent: Report — ✅ verified (N) / ⏳ awaiting your confirmation (M) / ❌ failed-or-untested (K)
```

## The four iron laws

1. **Local doesn't count** — compile / unit / type-check is sanity, not verification. The real environment is the only judge.
2. **Success ≠ correct** — `200` / `sent` / `committed` is *technical* success, not *business* correctness. Verify the **product**, not the call.
3. **No evidence, no "pass"** — every ✅ needs proof you can independently re-check (a log line, a DB row, a screenshot, a recipient's confirmation).
4. **Three tiers, never merged** — ✅ verified / ⏳ awaiting confirmation / ❌ failed-or-untested. Dressing ⏳ or ❌ up as ✅ is a deceptive report.

## The 8 phases

| # | Phase | Verifies |
|---|---|---|
| 0 | **Pre-flight** | The change actually reached the target environment (+ rollback ready) |
| 1 | **Logic** | Every branch — normal / boundary / illegal input — in the real environment |
| 2 | **Data accuracy** | Key numbers reconciled against an *independent source* |
| 3 | **Same-pattern sweep** | Fixed one spot? `grep` the whole codebase for the same pattern |
| 4 | **Independent review** | A *second, independent* AI (or adversarial self-review) hunts your blind spots |
| 5 | **Production E2E** | Real calls against the real environment; side effects verified to the **result** |
| 6 | **Human simulation** | A real browser/client walks the full user flow |
| 7 | **Sign-off** | Re-check every "pass" has real proof; emit the three-tier report |

Full details: **[PROTOCOL.md](PROTOCOL.md)**.

## Why trust this

Every rule here is the scar tissue of a real failure — an agent that reported "done" and was wrong. The **[Iron Laws](IRON-LAWS.md)** (R1–R15) each carry the story of the bug that created them: an email that said `sent` but never arrived, a permission migration that fixed one file and missed five, a unique index that got dropped but never rebuilt, a "second AI" that dropped its connection mid-review.

## Adopt it in your project

See **[ADOPT.md](ADOPT.md)**. The short version:

1. Drop [`PROTOCOL.md`](PROTOCOL.md) into your repo (e.g. `docs/proof-of-done.md`).
2. Paste [`templates/trigger-rule.md`](templates/trigger-rule.md) into your AI rules file (`CLAUDE.md` / `.cursorrules` / `.clinerules` / `AGENTS.md` / …).
3. Map the generic steps to your stack (deploy command, log query, browser tool) using the table in ADOPT.md.
4. Every time you catch a new "AI-reports-done-but-wrong" pattern, append it as a new Iron Law. The protocol is meant to evolve.

## License

[MIT](LICENSE) — use it, fork it, bend it to your stack.

---

## 中文文档

AI 协作开发最大的失败模式：**把「代码写完 + 编译过 + 单测过」当成「任务完成」**。这套协议把那些坑硬性写进流程，让 AI 没法偷懒，让你拿到的「通过」是真通过——每一条都有可独立复核的凭据。

- **[PROTOCOL.md](PROTOCOL.md)** — 完整 8 阶段流程（部署前置 → 逻辑 → 数据准确 → 同类排查 → 独立审查 → 生产端到端 → 模拟人类 → 收口）
- **[IRON-LAWS.md](IRON-LAWS.md)** — 踩坑沉淀的铁律 R1–R15，每条都有「背后那个真实翻车的 bug」
- **[ADOPT.md](ADOPT.md)** — 如何落地到你自己的项目（工具栈映射 + 触发词配置）
- **[templates/](templates/)** — 触发词规则模板 + 三档报告模板

### 一句话精神

> **本地通过不算数，接口成功不算数，没收到确认不算「通过」。报告分三档，不混档。触发词配进规则文件，AI 自动跑完，你只验最终报告。**
