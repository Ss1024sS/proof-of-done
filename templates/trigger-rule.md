# 触发词规则模板

把下面这段贴进你的 AI 规则文件（`CLAUDE.md` / `.cursorrules` / `.clinerules` / `AGENTS.md` / `.github/copilot-instructions.md` …）。触发词按团队习惯改。

---

```markdown
## 自动化行为规则：长期测试（proof-of-done）

用户说「启动长期测试」/「跑长期测试」/「走测试流程」（或你自定的触发词）时，
默认按 `docs/proof-of-done.md` 的 8 阶段（Phase 0–7）逐阶段全跑，不用再问。
前提：改动已实现完。

铁律（写在最前，每次必须遵守）：
- 本地编译 / 单测 / 类型检查不算验证，必须打到目标环境（生产 / staging）跑
- 接口 200 / status=sent / commit 完成 ≠ 结果正确，必须验「产物」
- 投递类副作用（邮件 / 通知 / webhook）必须接收方确认收到，否则标「⏳ 待确认」不标「✅」
- 修一处必 grep 全同类（Phase 3）：一个模块的端点常散在多个文件
- 第二个 AI 审查（Phase 4）不可用时改对抗性自审，不准跳过
- 测试期的对外副作用别打扰真实用户（mock 投递，但别动生产定时任务）
- 报告分三档：✅已验证 / ⏳待确认 / ❌未通过，绝不混档

8 阶段：0 部署前置 → 1 逻辑 → 2 数据准确 → 3 同类排查 →
        4 独立审查 → 5 生产 E2E → 6 模拟人类 → 7 收口。
```

---

## 各 AI 工具的规则文件位置

| AI 工具 | 规则文件 |
|--------|---------|
| Claude Code | `CLAUDE.md` |
| Cursor | `.cursorrules` |
| Cline | `.clinerules` |
| Aider | `.aider.conf.yml` / `CONVENTIONS.md` |
| GitHub Copilot | `.github/copilot-instructions.md` |
| Continue | `.continue/config.json`（system prompt） |
| Codex CLI | `AGENTS.md` / `CODEX.md` |
