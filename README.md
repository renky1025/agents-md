# agent-md

Claude Code / Codex 通用系统级 Agent 指令集。本仓库是 `~/.claude/` 下核心配置文件的版本化副本，用于跨机器同步、审阅与迭代。

## 文件说明

| 文件 | 作用 |
| --- | --- |
| `AGENTS.md` | 系统主指令（目标、沟通规范、最小实现原则、验证流程、记忆权威顺序） |
| `CLAUDE.md` | 指向 `AGENTS.md` 的符号链接，Claude Code 通过它加载同一份指令 |
| `agent_software_engineering_methodology_playbook.md` | 架构治理手册，写代码前必读，与 `AGENTS.md` 冲突时暂停并向用户确认 |
| `anti-ai-writing-style.md` | 写作风格规范，避免 AI 味输出 |
| `typescript.md` | TypeScript / 前端工程的语言级编码约定 |

`AGENTS.md` 的"渐进披露"章节按需引用上述文件，不在主文件中展开全文。

## 使用方式

将文件放回 Claude Code 配置目录：

```bash
cp AGENTS.md CLAUDE.md *.md ~/.claude/
```

或直接用本仓库作为配置的单一来源（推荐，便于 `git pull` 更新）：

```bash
ln -sf <repo>/AGENTS.md ~/.claude/AGENTS.md
ln -sf AGENTS.md ~/.claude/CLAUDE.md
ln -sf <repo>/agent_software_engineering_methodology_playbook.md ~/.claude/
ln -sf <repo>/anti-ai-writing-style.md ~/.claude/
ln -sf <repo>/typescript.md ~/.claude/
```

Codex 及其他工具直接读取 `AGENTS.md`，无需符号链接。

## 未包含的内容

- `~/claude-memory/memory-rules.md`：被 `AGENTS.md` 的"记忆权威"章节引用，属于独立的记忆仓库，不在本仓库范围内。
- 项目级 `AGENTS.md` / `CLAUDE.md`：随各项目仓库分发，不是系统级配置。

## 说明

仓库内容为个人配置，不含密钥、Token 或凭据。修改后请保持 `AGENTS.md` 与各子文件之间的引用关系一致。
