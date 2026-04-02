# claude-config

個人 Claude Code 配置工作區，用於追蹤自建 skills、MCP servers 及環境設定，方便跨機器同步。

## 目錄結構

```
claude-config/
├── CLAUDE.md              # Claude Code 操作指引
├── skills/                # 自建 skills
│   └── cursor-agent/      # Cursor Agent 無頭子代理 skill
└── README.md
```

---

## Skills 明細

### `cursor-agent`

> 將實作任務委派給 Cursor Agent 作為無頭子代理（headless subagent）。
> Cursor Agent 可自主讀取、寫入、修改專案檔案，適合用來平行或序列執行編程任務，同時不消耗 Claude Code 的 context window。

**安裝方式**（複製到 `~/.claude/skills/cursor-agent/SKILL.md`）：

```bash
cp skills/cursor-agent/SKILL.md ~/.claude/skills/cursor-agent/SKILL.md
```

**前置需求**：已安裝 [Cursor](https://www.cursor.com/) 並可使用 `cursor-agent` CLI。

**Binary 路徑（Windows）**：
```
%LOCALAPPDATA%\cursor-agent\agent.ps1
```

**觸發時機**：
- 需要平行執行多個獨立的實作任務
- 想保留 Claude Code context 給架構設計、review 等高層工作
- 有大量重複性程式碼需要生成（boilerplate、CRUD、元件）

**核心用法**：

```bash
pwsh -Command "
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
cd 'C:\path\to\your\project'
\$output = & \"\$env:LOCALAPPDATA\cursor-agent\agent.ps1\" \`
  '-p' '你的指令' \`
  '--trust' '--yolo' '--output-format' 'text' 2>&1
\$output | Out-File 'cursor-output.txt' -Encoding UTF8
"
cat cursor-output.txt
```

**主要 Flag**：

| Flag | 說明 |
|------|------|
| `-p "prompt"` | 無頭模式，帶入初始指令 |
| `--trust` | 信任工作目錄（首次或新目錄必須加） |
| `--yolo` / `-f` | 自動核准所有檔案寫入與 shell 指令 |
| `--plan` | 唯讀規劃模式（不寫入檔案） |
| `--mode ask` | 問答／解釋模式（不寫入檔案） |
| `--output-format text` | 純文字輸出（適合人工閱讀） |
| `--output-format json` | 結構化輸出（適合程式解析） |
| `--model claude-sonnet-4` | 指定模型（預設自動選擇） |
| `--continue` | 繼續上一個 session |
| `--resume <chatId>` | 恢復指定 session |

**協作模型**：

```
Claude Code（我）= 指揮官
  ├─ 設計架構與任務拆解
  ├─ 指派任務給 Cursor Agent
  ├─ 審查輸出、執行驗證（type-check、build）
  └─ 處理 git 操作（branch、commit、PR）

Cursor Agent = 實作者
  ├─ 撰寫 boilerplate 與重複性程式碼
  ├─ 根據明確規格實作模組
  └─ 一次可處理多個檔案
```

**最佳實踐**：
1. 給 Cursor Agent **明確的 context**：包含檔案路徑、函式名稱、要使用的型別
2. **一次一模組**：不要要求它實作整個系統
3. **執行後驗證**：跑 `npm run type-check` 與 `npm run build`
4. 複雜任務先用 `--plan`，確認計畫後再加 `--yolo`
5. **輸出到檔案**：長指令會被背景執行，寫入檔案再讀取

---

### `hims-code-archaeologist-skill`

> 對 .NET / ASP.NET / MSSQL 系統進行軟體考古（Software Archaeology）。
> 透過 7 個分析 Phase，系統性地發現技術債、架構漂移、依賴腐敗與死程式碼。
> **完全唯讀 — 不修改任何原始碼。**

**安裝方式**：
```bash
mkdir -p ~/.claude/skills/hims-code-archaeologist
cp skills/hims-code-archaeologist/SKILL.md ~/.claude/skills/hims-code-archaeologist/SKILL.md
```

**適用語言棧**：C# / .NET / ASP.NET / MSSQL  
**支援 OS**：Windows / Linux / macOS

**觸發時機**：
- 接手遺留 .NET 系統，需要評估風險
- 重構或現代化前的技術債盤點
- 找出無文件記錄的依賴關係
- 分析架構是否符合分層設計（Controllers → Services → Domain → Infrastructure）

**7 個分析 Phase**：

| Phase | 名稱 | 主要工具 | 說明 |
|-------|------|---------|------|
| 0 | Dependency Check | bash | 檢測環境工具，缺工具時降級而非中止 |
| 1 | Historical Excavation | git | commit 頻率、hotfix 率、高 churn 檔案 |
| 2 | Dependency Archaeology | dotnet CLI + rg | NuGet 過期/廢棄套件、循環 ProjectReference |
| 3 | Structural Decay | python3 + jscpd | 大型類別、長方法（>50行）、重複程式碼 |
| 4 | Architecture Drift | rg | 層次違規（Domain 引用 Controller）、Controller 直連 DB |
| 5 | Test Coverage | dotnet test + coverlet | 無測試的複雜檔案、測試壞味道（過度 mock、flaky） |
| 6 | Documentation Decay | git + jq | 過時 TODO、config key 是否被引用、文件新鮮度 |
| 7 | Dead Code | Roslyn + python3 + rg | 孤立型別、註解掉的程式碼、未被呼叫的 SQL 物件 |

**依賴工具**（缺少時會降級，不會中止）：

| 工具 | 用途 | Windows 安裝 |
|------|------|-------------|
| `git` | 所有歷史分析（必要） | 通常已安裝 |
| `dotnet` | NuGet 分析（必要） | dotnet.microsoft.com |
| `rg` (ripgrep) | 程式碼搜尋（建議） | `winget install BurntSushi.ripgrep.MSVC` |
| `python3` | 循環依賴、死程式碼腳本（建議） | `winget install Python.Python.3` |
| `jq` | JSON config 分析（選用） | `winget install jqlang.jq` |
| `jscpd` | 重複程式碼偵測（選用） | `npm install -g jscpd` |
| `coverlet` | 測試覆蓋率（選用） | `dotnet tool install -g coverlet.console` |

> **缺少 python3 的 script？** Claude 會在執行時臨時生成再執行，不需要事先準備。

---

## 快速同步到新機器

```bash
git clone https://github.com/Ikeli0320/claude-config.git
cd claude-config

# 安裝所有自建 skills
mkdir -p ~/.claude/skills/cursor-agent
cp skills/cursor-agent/SKILL.md ~/.claude/skills/cursor-agent/SKILL.md

mkdir -p ~/.claude/skills/hims-code-archaeologist
cp skills/hims-code-archaeologist/SKILL.md ~/.claude/skills/hims-code-archaeologist/SKILL.md
```
