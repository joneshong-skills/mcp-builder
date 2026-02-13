[English](README.md) | [繁體中文](README.zh.md)

# mcp-builder

一個 Claude Code 技能，引導建立高品質的 MCP (Model Context Protocol) 伺服器，支援 Python 和 TypeScript。

## 說明

此技能提供完整的 MCP 伺服器開發流程，將 API 或服務封裝為可被 LLM agent 發現和使用的工具。涵蓋完整開發生命週期：API 研究與規劃、最佳實踐實作、程式碼審查、測試與評估。

## 功能特色

- 引導以工作流程為導向的工具設計（而非單純的 API 端點封裝）
- 提供 Python (FastMCP) 和 TypeScript (MCP SDK) 的語言專屬實作指南
- 包含命名、回應格式、分頁、傳輸協議和安全性的最佳實踐
- 附帶可直接使用的 MCP 連線處理和伺服器評估腳本
- 涵蓋工具標註（`readOnlyHint`、`destructiveHint`、`idempotentHint`、`openWorldHint`）
- 包含評估框架，用於測試 MCP 伺服器與 LLM agent 的整合品質

## 安裝

將技能目錄複製到 Claude Code 技能資料夾：

```
cp -r mcp-builder ~/.claude/skills/
```

放置在 `~/.claude/skills/` 的技能會被 Claude Code 自動發現，無需額外註冊。

## 使用方式

透過要求 Claude Code 建立 MCP 伺服器來觸發此技能。範例提示：

- 「幫我建立一個 GitHub API 的 MCP server」
- 「寫一個包裝 Slack 的 MCP 伺服器」
- 「幫我做一個資料庫的 MCP 工具」
- 「用 Python 建立一個 Jira 的 MCP server」

Claude 會自動執行研究、實作、審查和評估等階段。
