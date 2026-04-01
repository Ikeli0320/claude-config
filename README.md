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

## 快速同步到新機器

```bash
git clone https://github.com/Ikeli0320/claude-config.git
cd claude-config

# 安裝所有自建 skills
mkdir -p ~/.claude/skills/cursor-agent
cp skills/cursor-agent/SKILL.md ~/.claude/skills/cursor-agent/SKILL.md
```
