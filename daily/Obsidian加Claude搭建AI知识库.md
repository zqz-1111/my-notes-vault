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

### Git 切换 SSH

HTTPS 推送在国内网络不稳定，改成了 SSH：
- 本地已有 SSH key（ed25519）
- `git remote set-url origin git@github.com:zqz-1111/my-notes-vault.git`
- 推送稳定了，后续由 Claude 直接 push，不依赖 Obsidian Git 插件

### 打通 IDEA + VSCode 双端 Claude

目标：IDEA 学 Java 项目时也能直接写笔记到 Obsidian。

1. 在 IDEA 里安装了 Claude Code 插件
2. 发现两边的 Claude 实例不共享记忆
3. 解决方案：在 vault 根目录建 `CLAUDE.md`（工作流指令），任何 Claude 实例启动时自动读取
4. 更进一步：把指令写到 `~/.claude/CLAUDE.md`（全局），这样不管在哪个文件夹打开 Claude 都知道流程

### 建立 Vault 目录结构

```
MyVault/
├── CLAUDE.md           ← 工作流指令
├── daily/              ← 日记/杂记
├── projects/           ← 项目学习笔记
├── snippets/           ← 代码片段库
├── interview/          ← 面试题收集
├── progress/           ← 学习进度追踪
├── resources/          ← 参考资料
└── assets/             ← 附件
```

### 编写完整工作流 CLAUDE.md

给 Claude 写了一套完整的自动化指令，包含 6 大能力：

1. **新对话自动恢复** — 读 `progress/学习进度.md`，接上上次学到哪
2. **项目上下文记忆** — 记住当前在学什么项目
3. **自动判断写几篇** — 回顾对话按主题拆分，同主题追加不重复
4. **多目录分发** — 项目笔记/代码片段/面试题自动分类存储
5. **Wikilinks 关联** — 笔记间用 `[[笔记名]]` 互相链接，形成知识图谱
6. **主动提醒** — 学了很多没记笔记时主动问"要记一下吗？"
7. **代码变更记录** — 记录增删改了哪些文件

### 最终链路

```
IDEA 学习 → Claude 参与改错/写代码 → 说"写笔记"
    ↓
Claude 回顾对话 → 自动判断主题 → 查重 → 分类 → 命名 → 写入
    ↓
Obsidian vault（daily/ + projects/ + snippets/ + interview/）
    ↓
git push → GitHub 私有仓库
```

---

> 今天从零搭建了一套完整的 AI 知识管理系统：MCP 打通读写、Git 版本管理、GitHub 备份、双端 Claude 共享指令、自动化笔记工作流。明天开始用它记录 Java 项目学习。
