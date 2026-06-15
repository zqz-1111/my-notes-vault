---
date: '2026-06-14'
tags:
  - daily
  - obsidian
  - claude-code
  - mcp
  - setup
  - git
  - github
  - backup
---
# 2026-06-14 周六

## 今日记录：打通 Claude Code 与 Obsidian

今天完成了 Claude Code 通过 MCP 协议直接读写 Obsidian vault 的配置。

### 做了什么

1. **了解背景** — 研究了 Obsidian + Claude AI 的搭配，发现这个组合最近很火，核心来自 Karpathy 的 LLM Wiki 概念：让 AI 帮你做知识管理的 bookkeeping（整理、链接、归档），人负责思考和产出
2. **安装 mcpvault** — 用 `npm install -g @bitbonsai/mcpvault` 安装了 MCP server，零依赖，不需要 Obsidian 插件
3. **配置 MCP** — 在 `~/.claude.json` 的 `mcpServers` 中添加了 obsidian 配置，指向 vault 路径 `F:\obsidion\ObsidianVault\MyVault`
4. **重启生效** — Reload Window 后 Claude Code 成功加载了 14 个 obsidian 工具，可以读写笔记、搜索、管理标签等

### 踩的坑

- `mcpvault` 全局安装后命令不在 PATH 里，需要用 `cmd /c npx` 的方式调用
- 配置完需要完全重启 VSCode 窗口（Ctrl+Shift+P → Reload Window）才能加载 MCP server

### 核心思路

```
人的工作：阅读、思考、产出洞察
AI 的工作：整理、链接、维护知识库
```

Karpathy 的比喻很精准：**Obsidian 是 IDE，LLM 是工程师，wiki 是代码库。**

### 三个核心操作

| 操作 | 说明 |
|---|---|
| Ingest（摄入） | 新资料喂给 AI，自动提取重点、写成结构化笔记、更新交叉引用 |
| Query（查询） | 基于积累的笔记合成答案，好的回答存回 wiki |
| Lint（健检） | 定期扫描 vault，找矛盾、过时内容、孤立页面 |

### 配置 Git 版本管理 + GitHub 远程备份

MCP 跑通后，又把 vault 接上了 Git，实现自动版本管理和云端备份：

1. **vault 初始化 Git** — `git init` + 建 `.gitignore`（排除 workspace.json 等临时文件）
2. **安装 Obsidian Git 插件** — 社区插件市场直接搜，免费开源
3. **创建 GitHub 私有仓库** — `zqz-1111/my-notes-vault`，笔记不会公开
4. **首次 push** — 所有笔记已推到 GitHub
5. **配置自动备份** — 插件每 10 分钟自动 commit + push，打开 Obsidian 自动 pull

踩的坑：插件提示 "Git is not ready"，需要手动指定 Git 路径 `F:\git\Git\cmd\git.exe`

现在的完整链路：

```
Obsidian 笔记 ←→ Git 本地仓库 ←→ GitHub 私有仓库
                    ↑
              Obsidian Git 插件（自动 commit + push）
```

### 接下来可以做的

- [ ] 在 vault 根目录建立 `CLAUDE.md`，写上笔记规范和 AI 行为边界
- [ ] 设置 `/today` slash command，自动生成每日笔记
- [ ] 试试批量给笔记加标签和反向链接
- [ ] 建立 PARA 结构的文件夹（projects / areas / resources / archive）

---

> 今天最大的收获：从"知道 Obsidian + Claude 很火"到"自己跑通了整个链路"。下一步是把这套工具真正用起来，让知识库开始积累。
