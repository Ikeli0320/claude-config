# CLAUDE.md

本檔案提供 Claude Code（claude.ai/code）在此 repository 中的操作指引。

## 用途

這是個人的 Claude CLI / Claude Code 配置與文件工作區，用於追蹤已安裝的 skills、MCP servers、插件及相關設定，以便在多台開發機器間同步環境。

本 repository **不包含**應用程式原始碼，無建置系統、測試套件或 lint pipeline。

## 內容

- 已安裝的 Claude Code skills、插件、MCP servers 的文件與說明
- 用於在新機器上重建 Claude Code 環境的配置參考與安裝指南
- 任何為跨機器可攜性建立的輔助腳本或範本

## 關鍵外部路徑（Windows）

| 路徑 | 說明 |
|------|------|
| `~/.claude/settings.json` | 全域 Claude Code 設定（權限、更新頻道） |
| `~/.claude/skills/` | 已安裝的 skills（每個 skill 含 SKILL.md） |
| `~/.claude/plugins/` | 已安裝的插件與 marketplace 快取 |
| `~/.claude/projects/` | 各專案的 session 資料與 memory |

## 慣例

- **語言**：使用者以繁體中文溝通，除非是程式碼或英文文件，否則請以相同語言回應。
- **平台**：Windows 11，使用 Git Bash — 採用 Unix 風格路徑與 shell 語法。
- **配置文件**：記錄設定時，請標明所屬範疇（全域 `settings.json` vs 專案層級 `settings.local.json`）。
