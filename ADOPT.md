# 落地到你自己的项目

四步把这套协议接进任何项目、任何 AI 工具。

---

## Step 1 — 把协议放进项目

```
your-project/
├─ docs/
│  └─ proof-of-done.md      ← 复制本仓库 PROTOCOL.md (或直接引用本仓库链接)
├─ raw/                     ← 用户给的原始数据归档 (见 R4)
└─ <AI 规则文件>            ← Step 3 的触发词写这里
```

你可以直接 `git submodule` 引入本仓库，或把 `PROTOCOL.md` + `IRON-LAWS.md` 复制进 `docs/`。推荐复制 —— 因为你会基于自己项目的踩坑**持续往 IRON-LAWS 里加铁律**（Step 4）。

---

## Step 2 — 把通用步骤映射到你的工具栈

协议里的步骤是抽象的，照下表填成你公司的具体命令：

| 通用步骤 | 可能用的工具（任选） |
|---------|-------------------|
| **部署** | `docker compose up` / `kubectl rollout` / Helm upgrade / Vercel·Netlify deploy / PaaS push |
| **重启服务** | `docker compose restart` / `kubectl delete pod` / `systemctl restart` / 平台重启 |
| **反向代理 reload** | `nginx -s reload` / Ingress reload / ALB 刷新 / API Gateway redeploy |
| **环境内调函数** | `docker exec` / `kubectl exec` / SSH / serverless invoke |
| **查日志** | `docker compose logs` / `kubectl logs` / Loki / CloudWatch / Datadog |
| **查数据库** | `docker exec psql` / 云控制台 / DBeaver / TablePlus |
| **邮件验证** | 收件人确认 / IMAP 收信 / SendGrid·SES·Postmark dashboard |
| **通知验证** | Slack / 钉钉 / 企业微信 回执或机器人确认 |
| **独立 AI 审查** | 第二个 AI CLI（Codex / Claude / Gemini / Aider …） |
| **浏览器实测** | Playwright / Cypress / Puppeteer / 真浏览器 + DevTools |
| **自签证书绕过** | Chrome 证书页输入 `thisisunsafe`；或给内网域名配可信证书 |

把这张表填好放进 `docs/proof-of-done.md` 顶部，AI 跑流程时就知道每一步在你这儿具体怎么做。

---

## Step 3 — 把触发词写进 AI 规则文件

各 AI 工具的规则文件位置：

| AI 工具 | 规则文件 |
|--------|---------|
| Claude Code | `CLAUDE.md`（项目根） |
| Cursor | `.cursorrules` |
| Cline | `.clinerules` |
| Aider | `.aider.conf.yml` 或 `CONVENTIONS.md` |
| GitHub Copilot | `.github/copilot-instructions.md` |
| Continue | `.continue/config.json`（system prompt） |
| Codex CLI | `AGENTS.md` 或 `CODEX.md` |

把 [templates/trigger-rule.md](templates/trigger-rule.md) 的内容贴进去。核心是：**一个触发词 → 指向 `docs/proof-of-done.md` → AI 自动跑全套**，你不用每次手动指挥「再去验证一下数据库」「再去看下日志」。

---

## Step 4 — 持续沉淀教训

每次实战发现新的「AI 报完成但翻车」模式 → 立刻补进 `IRON-LAWS.md` 一条新铁律（`R16`、`R17`…）。

这是范式「长期」的真义：**它不是一份写死的清单，是一个随你踩坑持续生长的知识库**。R1–R15 就是这么一条条长出来的。

---

## 你需要自己拍板的设计选项

| 选项 | 影响 |
|------|------|
| 触发词用中文还是英文 | 团队语言习惯（如「启动长期测试」/「run the gauntlet」） |
| 报告是否留档 | 留档便于复盘，但增加工作量（建议 `reports/<date>-<topic>.md`） |
| 独立审查用哪两个 AI | 看你订阅了哪些 |
| 自动化烟测覆盖多少 | 覆盖越多越省人，但维护成本高 |
| 测试数据是否进 git | 看保密策略 + 文件体积（敏感数据建议 `.gitignore`） |
| Phase 6 浏览器测试是否每次都做 | 后端纯改动可跳；UI 改动**必做** |
| Phase 4 独立审查最大轮数 | 视复杂度，常 3–5 轮 |

---

## 与「写死版」的区别（通用化决策）

这套协议刻意把一切环境相关的东西抽象掉了，方便任何项目用：

| 写死版会写 | 通用版抽象成 |
|----------|-----------|
| `docker compose restart backend + nginx -s reload` | 「部署 / 重启 / 反代刷新」，你按工具栈填 |
| 某家 SMTP + IMAP 服务器地址 | 「投递类副作用必须接收方确认」 |
| 某个具体 AI CLI | 「用第二个独立 AI 审查」，任选 |
| 某个生产 IP / 域名 | 「生产 / Staging / Demo 环境」 |
| 某个签 token 函数 + 测试账号 | 「鉴权策略」三档优先级 |

> 一句话：**环境细节你填，方法论我给。**
