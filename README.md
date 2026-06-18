# proof-of-done

> **Local pass ≠ verified. HTTP 200 ≠ correct result. No receipt ≠ "done".**
> A tool-agnostic, domain-agnostic verification protocol for AI-assisted development — so that "done" means *provably done*, not "the code compiles and the tests are green."

**[English](#english)** · **[中文](#中文)** — 正文文档（PROTOCOL / IRON-LAWS / ADOPT）为英文 / docs are in English.

[The Protocol](PROTOCOL.md) · [Iron Laws](IRON-LAWS.md) · [Adopt it](ADOPT.md) · [Sync & iterate](SYNC.md) · [Templates](templates/) · version [`2026.06.18.2`](VERSION) (R1–R28)

---

<a name="english"></a>
## English

### The problem this solves

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

### How it works in one line

Register a **trigger word** in your AI rules file → say it once after a change → the agent runs all **8 phases** → you only review the final **three-tier report**.

```
You: "run the gauntlet"
Agent: [0 deploy → 1 logic → 2 data → 3 same-pattern sweep →
        4 independent review → 5 prod E2E → 6 real browser → 7 sign-off]
Agent: Report — ✅ verified (N) / ⏳ awaiting your confirmation (M) / ❌ failed-or-untested (K)
```

### The four iron laws

1. **Local doesn't count** — compile / unit / type-check is sanity, not verification. The real environment is the only judge.
2. **Success ≠ correct** — `200` / `sent` / `committed` is *technical* success, not *business* correctness. Verify the **product**, not the call.
3. **No evidence, no "pass"** — every ✅ needs proof you can independently re-check (a log line, a DB row, a screenshot, a recipient's confirmation).
4. **Three tiers, never merged** — ✅ verified / ⏳ awaiting confirmation / ❌ failed-or-untested. Dressing ⏳ or ❌ up as ✅ is a deceptive report.

### The 8 phases

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

### Why trust this

Every rule here is the scar tissue of a real failure — an agent that reported "done" and was wrong. The **[Iron Laws](IRON-LAWS.md)** (R1–R24) each carry the story of the bug that created them: an email that said `sent` but never arrived; a permission migration that fixed one file and missed five; a unique index that got dropped but never rebuilt (because idempotency was based on a stale cache); a "second AI" that dropped its connection mid-review; a `Query(None)` default that was a truthy object in direct-call tests; a `?back=` param that was an open redirect; a green `tsc` that still failed the `next build` prerender.

### One methodology everywhere, best-fit anywhere

The base in this repo is the single upstream; each project keeps a thin **overlay**
(its stack-mapping table + its own laws) pinned to a base [`VERSION`](VERSION).
**[SYNC.md](SYNC.md)** defines the loop that keeps them consistent: pull the base
before a run; after a run, send *generic* scars upstream (a new iron law here) and
*project-specific* scars to the overlay. That is how every session shares the same
updated methodology while each project's test stays tuned to itself.

### Adopt it in your project

See **[ADOPT.md](ADOPT.md)**. The short version:

1. Drop [`PROTOCOL.md`](PROTOCOL.md) into your repo (e.g. `docs/proof-of-done.md`).
2. Paste [`templates/trigger-rule.md`](templates/trigger-rule.md) into your AI rules file (`CLAUDE.md` / `.cursorrules` / `.clinerules` / `AGENTS.md` / …).
3. Map the generic steps to your stack (deploy command, log query, browser tool) using the table in ADOPT.md.
4. Every time you catch a new "AI-reports-done-but-wrong" pattern, append it as a new Iron Law. The protocol is meant to evolve.

### License

[MIT](LICENSE) — use it, fork it, bend it to your stack.

---

<a name="中文"></a>
## 中文

> 正文流程文档（[PROTOCOL](PROTOCOL.md) / [IRON-LAWS](IRON-LAWS.md) / [ADOPT](ADOPT.md)）是英文，方便国际复用；这一节给中文读者完整讲清。

### 这套协议解决什么

AI 协作开发最大的失败模式：**把「代码写完 + 编译过 + 单测过」当成「任务完成」**。但几乎从来不是。

| AI 报「完成」 | 实际可能是 |
|---|---|
| 本地单测全绿 | 生产连不上真实数据库 / 数据是错的 |
| HTTP 接口返回 `200` | 业务结果错（字段值不对 / 计算偏差） |
| 邮件接口 `status=sent` | 收件人邮箱根本没收到 |
| 写库 `INSERT` commit | 落库字段值跟预期不一致 |
| 文件下载成功 | 内容损坏或为空 |
| 前端编译过 | 浏览器报红错 / 数据不显示 |

**proof-of-done** 把这些坑硬性写进流程，AI 没法偷懒，你拿到的「通过」是真通过 —— 每条都有可独立复核的凭据。

### 一句话怎么用

在 AI 规则文件里登记一个**触发词** → 改完说一句 → AI 自动跑完 **8 阶段** → 你只验最终的**三档报告**。

### 四条核心原则

1. **本地不算数** —— 编译 / 单测 / 类型检查只是 sanity，不算验证。真实环境才是唯一裁判。
2. **接口成功 ≠ 结果正确** —— `200` / `sent` / `committed` 只是*技术*成功，不是*业务*正确。验「产物」，不是验「调用」。
3. **没真凭据不写「通过」** —— 每条 ✅ 都要有能独立复核的证据（日志行 / 查库结果 / 截图 / 收件方确认）。
4. **报告分三档不混** —— ✅已验证 / ⏳待确认 / ❌未通过。把 ⏳ 或 ❌ 包装成 ✅ 是欺骗性报告。

### 8 阶段

| # | 阶段 | 验什么 |
|---|---|---|
| 0 | **部署前置** | 改动真进了目标环境（+ 回滚就绪） |
| 1 | **逻辑** | 真实环境里走遍每个分支：正常 / 边界 / 非法输入 |
| 2 | **数据准确** | 关键数值跟*独立来源*对账 |
| 3 | **同类排查** | 修了一处？`grep` 全代码库的同类 pattern |
| 4 | **独立审查** | 第二个*独立* AI（或对抗性自审）抓你的盲区 |
| 5 | **生产端到端** | 真打真实环境；副作用验证到**结果** |
| 6 | **模拟人类** | 真浏览器 / 客户端走完整用户流程 |
| 7 | **收口** | 复查每条「通过」是否有真凭据；出三档报告 |

完整细节看 **[PROTOCOL.md](PROTOCOL.md)**（英文）。

### 为什么可信

这里每一条规则都是一次真实翻车的疤 —— AI 报了「完成」，但实际是错的。**[铁律](IRON-LAWS.md)** R1–R24 每条都带「背后那个 bug 的故事」：一封说 `sent` 却没到的邮件；一次只改一个文件、漏了同模块五处的权限迁移；一个被 DROP 了却没重建的唯一索引（幂等判断用了过期缓存）；一个审查到一半掉线的「第二个 AI」；一个在直调单测里是 truthy 对象的 `Query(None)` 默认值；一个成了开放重定向的 `?back=` 参数；一个 `tsc` 全绿却挂在 `next build` 预渲染的页面。

### 一份方法论，处处一致，又各自最优

本仓库的基线是唯一上游；每个项目保留一份很薄的**特化层**（它的工具映射表 + 它自己的铁律），
钉住某个基线 [`VERSION`](VERSION)。**[SYNC.md](SYNC.md)** 定义了让两者保持一致的环：跑测前先拉基线；
跑完后，把*通用*的疤送回上游（在这里加一条新铁律），把*项目特定*的疤留在特化层。这样每个 session
都共享同一份更新后的方法论，而每个项目的长测又各自贴合自己。

### 落地到你的项目

看 **[ADOPT.md](ADOPT.md)**。简版：

1. 把 [`PROTOCOL.md`](PROTOCOL.md) 放进你的仓库（如 `docs/proof-of-done.md`）
2. 把 [`templates/trigger-rule.md`](templates/trigger-rule.md) 贴进你的 AI 规则文件（`CLAUDE.md` / `.cursorrules` / `.clinerules` / `AGENTS.md` …）—— 触发词可以用中文（如「启动长期测试」）
3. 按 ADOPT.md 的表把通用步骤映射成你的具体命令（部署 / 查日志 / 浏览器工具）
4. 每抓到一个新的「AI 报完成但翻车」模式，就加一条新铁律。协议是用来生长的。

### 许可

[MIT](LICENSE) —— 随便用、fork、改成你的工具栈。

---

> **本地通过不算数，接口成功不算数，没收到确认不算「通过」。报告分三档，不混档。触发词配进规则文件，AI 自动跑完，你只验最终报告。**
