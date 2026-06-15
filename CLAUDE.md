# Obsidian Vault 工作指南

## 身份
这是一个个人学习知识库，主人在学习黑马程序员的 Java 项目课程。

## 核心工作流

当用户说"总结今天的学习"或类似指令时，执行以下流程：

1. 根据今天的对话内容，用中文写一篇学习笔记
2. 笔记内容包括：学了什么、写了什么代码、遇到的问题及解决方案、明天计划
3. 用 MCP 工具 `write_note` 写入 `daily/` 目录
4. 文件名用内容概括，**不要用日期**（例：`SpringBoot整合MyBatis.md`）
5. 写完后执行 `git add -A && git commit -m "描述" && git push`

## 文件名规范

- 日记/学习笔记：用中文概括内容，如 `黑马JDBC学习笔记.md`
- 不要用 `2026-06-14.md` 这种日期格式

## Git 推送

vault 已关联 GitHub SSH 仓库：`git@github.com:zqz-1111/my-notes-vault.git`

每次写完笔记后自动 commit + push。

## Vault 结构

- `daily/` — 日常学习笔记和日记
- `assets/` — 附件
