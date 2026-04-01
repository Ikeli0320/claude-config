---
name: doc-to-hybrid-html
description: 將 PDF、Word、PPT、圖片等文件轉換為「混合式 HTML」——UI 截圖頁用 HTML/CSS 重建（可編輯），程式碼頁用 <pre><code> 語法高亮，純文字頁用語意 HTML。結果為單一自含 HTML 檔案，完全可編輯且不依賴任何外部圖片。
---

# 文件轉混合式 HTML (doc-to-hybrid-html)

將 PDF / Word / PPT / 圖片等不可編輯文件轉換為**完全可編輯的單一 HTML 檔案**。核心策略是**混合式重建**：根據每頁的內容類型選用最省 token、最高可編輯性的方案。

---

## When to Use

- 需要讓靜態文件內容可以直接在 HTML 中修改（UI 截圖、表格、程式碼都要能編輯）
- 需要可供網頁瀏覽的技術整合文件、SDK 手冊、操作指南
- 原始 PDF/Word/PPT 已難以取得或工具版本過舊，無法直接編輯
- 文件含有大量 macOS / Xcode / IDE 截圖需要保留視覺語意

---

## How It Works

### 第一步：取得可讀格式

**PDF**：
```bash
# 安裝 poppler（macOS）
brew install poppler

# 將所有頁面轉為 PNG（200 DPI 清晰度足夠 OCR + 視覺分析）
pdftoppm -png -r 200 "文件.pdf" "輸出目錄/page"
# 產出：page-01.png, page-02.png, ...
```

**Word / PPT / 圖片**：直接以 Read tool 讀取（Claude Code 原生支援 docx、pptx、png、jpg）。

---

### 第二步：頁面分類（關鍵決策）

逐頁掃描後，依內容類型分入三個桶：

| 類型 | 判斷標準 | 轉換方案 |
|------|---------|---------|
| **UI 截圖頁** | 截圖佔頁面 >30%，含 IDE / OS / 瀏覽器 UI | **方案 C** — HTML/CSS 重建 |
| **程式碼頁** | 程式碼佔頁面 >50% | **方案 B** — `<pre><code>` + 語法高亮 |
| **純文字/表格頁** | 文字、列表、表格為主 | **方案 A** — 語意 HTML（`<h2>`, `<table>`, `<ul>`） |

> **經驗法則（以 65 頁 iOS SDK 文件為例）**：
> UI 截圖頁 ~15 頁（方案 C）、程式碼頁 ~35 頁（方案 B）、純文字頁 ~15 頁（方案 A）

---

### 第三步：Token 成本估算

選擇方案前先評估成本：

| 方案 | 相對 token 成本 | 可編輯性 |
|------|----------------|---------|
| 純 PNG 嵌入 | 1× (基準) | 不可編輯 |
| 方案 A 語意 HTML | ~1.5× | 完全可編輯 |
| 方案 B `<pre><code>` | ~2× | 完全可編輯 |
| 方案 C HTML/CSS 重建 | ~8–12× | 完全可編輯 |
| **混合 A+B+C** | **~4–5×** | **完全可編輯** |

> 方案 C 只用在真正有視覺語意的 UI 頁，其餘頁面用 A/B，可將總成本壓在 4–5× 範圍。

---

### 第四步：HTML/CSS 元件庫（方案 C 必備）

以下是常用 macOS / Xcode UI 的 CSS 片段，可直接複製進 `<style>` 區塊：

#### macOS Finder 視窗
```css
.finder-window { background:#fff; border:1px solid #c0c0c0; border-radius:8px; box-shadow:0 4px 16px rgba(0,0,0,.15); overflow:hidden; }
.finder-titlebar { background:linear-gradient(#e8e8e8,#d4d4d4); padding:8px 12px; display:flex; align-items:center; gap:8px; }
.finder-dots span { width:12px; height:12px; border-radius:50%; display:inline-block; }
.finder-dots span:nth-child(1){background:#ff5f57;} .finder-dots span:nth-child(2){background:#febc2e;} .finder-dots span:nth-child(3){background:#28c840;}
.finder-row.selected { background:#0064d2; color:#fff; }
```

#### Xcode IDE（General / Build Settings / Build Phases）
```css
.xcode-window { background:#1e1e1e; border-radius:8px; overflow:hidden; font-family:'SF Mono',monospace; color:#d4d4d4; }
.xcode-sidebar { background:#252526; border-right:1px solid #3e3e42; padding:8px 0; min-width:200px; }
.xcode-tabs { background:#2d2d30; display:flex; }
.xcode-tab { padding:6px 16px; color:#9d9d9d; cursor:pointer; }
.xcode-tab.active { background:#1e1e1e; color:#fff; border-top:2px solid #007acc; }
.xcode-kv-row { display:flex; padding:4px 8px; border-bottom:1px solid #3e3e42; }
.xcode-kv-row .key { flex:1; color:#9cdcfe; } .xcode-kv-row .val { color:#ce9178; }
```

#### macOS 開啟檔案對話框
```css
.macos-dialog { background:#f5f5f5; border:1px solid #aaa; border-radius:10px; padding:16px; box-shadow:0 8px 24px rgba(0,0,0,.3); }
.dialog-file-list { background:#fff; border:1px solid #ddd; border-radius:6px; max-height:200px; overflow-y:auto; }
.file-row { padding:4px 12px; cursor:pointer; } .file-row.selected { background:#0064d2; color:#fff; }
.dialog-actions { display:flex; justify-content:flex-end; gap:8px; margin-top:12px; }
.btn-cancel { background:#e0e0e0; border:1px solid #bbb; border-radius:6px; padding:4px 16px; }
.btn-confirm { background:#007aff; color:#fff; border:none; border-radius:6px; padding:4px 16px; }
```

#### 標注框（Note / Warning）
```css
.note { background:#fffde7; border-left:4px solid #f9a825; padding:8px 12px; margin:8px 0; border-radius:0 4px 4px 0; }
.warning { background:#fdecea; border-left:4px solid #c62828; padding:8px 12px; margin:8px 0; border-radius:0 4px 4px 0; }
```

---

### 第五步：語法高亮（方案 B）

程式碼區塊使用 `<span>` 標記語法類別，**不引入任何外部 JS 函式庫**：

```css
/* 放入 <style> */
pre { background:#1e1e1e; color:#d4d4d4; padding:16px; border-radius:6px; overflow-x:auto; }
.kw { color:#c792ea; font-weight:bold; }   /* keywords: func, class, if, return */
.cm { color:#546e7a; font-style:italic; }  /* comments */
.str { color:#f07178; }                     /* string literals */
.type { color:#82aaff; }                    /* type names */
.fn { color:#82aaff; }                      /* function names */
.todo { color:#ff5370; font-weight:bold; }  /* TODO / FIXME markers */
```

HTML 範例：
```html
<pre><code><span class="kw">func</span> <span class="fn">viewDidLoad</span>() {
    <span class="cm">// 初始化 SDK</span>
    <span class="type">HotaiSdkManager</span>.shared.start()
}</code></pre>
```

---

### 第六步：文件結構

輸出的單一 HTML 檔案建議結構：

```html
<!DOCTYPE html>
<html lang="zh-TW">
<head>
  <meta charset="UTF-8">
  <title>文件標題</title>
  <style>
    /* 1. Reset + 全域字體 */
    /* 2. 頁面容器與 TOC */
    /* 3. 方案 A 語意元素（h2, table, ul） */
    /* 4. 方案 B 程式碼語法高亮 */
    /* 5. 方案 C UI 元件（Finder, Xcode, Dialog…） */
    /* 6. Note / Warning 標注框 */
  </style>
</head>
<body>
  <!-- TOC -->
  <nav id="toc">…</nav>

  <!-- 各章節，id="section-X" 供 TOC 錨點連結 -->
  <section id="section-1">…</section>
</body>
</html>
```

---

## Examples

### 案例：和泰汽車 LoginSDK iOS 整合文件（65 頁 PDF）

**輸入**：65 頁 PDF，含 Finder 截圖、Xcode 截圖、Swift/ObjC 程式碼

**分類結果**：
- P1–4：封面、系統需求、對照表 → 方案 A
- P5–18：Finder、Xcode General/Build Settings/Build Phases/Signing 截圖 → 方案 C
- P19–53：Swift + ObjC AppDelegate、ViewController、SDK 呼叫 → 方案 B
- P54–65：SceneDelegate 表格 + Info.plist XML + Swift/ObjC → 方案 A+B

**輸出**：`完整版.html` — 3,080 行、168KB、完全自含、含 TOC 錨點

**實際轉換指令**：
```bash
# 1. PDF → PNG
pdftoppm -png -r 200 "和泰汽車_LoginSDK_iOS_整合文件_2026.pdf" "images/page"

# 2. 逐頁用 Read tool 讀取 PNG，分類後按方案輸出 HTML 片段
# 3. 組合成單一 HTML 檔案
```

---

## Common Pitfalls

1. **UI 截圖遺漏**：每一頁都要確認是否有截圖，不能只看文字。例如 Xcode General tab 的「Frameworks, Libraries, and Embedded Content」區塊容易被忽略。
2. **ObjC 語法高亮**：`#import`、`@interface`、`@implementation` 等 ObjC 特有語法要在 CSS 類別中單獨處理（`.directive` 類別），否則高亮不準確。
3. **中文字體**：`font-family` 記得加 `"PingFang TC", "Microsoft JhengHei"` 以確保繁體中文正確顯示。
4. **TOC 錨點**：每個 `<section>` 都加 `id`，TOC 使用 `<a href="#section-X">` 而非 JS scroll。
5. **token 預算超支**：方案 C 每頁約消耗 800–1200 output tokens；若預算有限，優先對含操作步驟的 UI 頁使用方案 C，單純示意圖改用方案 B 的 ASCII art 或省略。
