---
title: Claude Code 設定檔說明：settings.json、CLAUDE.md、Hooks 與 Subagents
date: 2026-07-30
updated:
categories:
tags:
  - Claude Code
cover:
---

把 **Claude Code** 用在日常開發中，它的設定系統其實比想像中豐富，除了最基本的 `settings.json`，還有 `CLAUDE.md`、hooks、MCP、subagents 等好幾種設定機制，各自負責不同的事情。這篇文章整理這些設定檔的用途、放置位置與常見寫法，作為日後查閱的筆記。

在進入細節之前，先用一張表快速了解這幾種機制的定位，避免混為一談：

| 類型 | 作用 |
| --- | --- |
| **settings.json** | 技術層面的強制設定，如權限規則、環境變數、模型選擇 |
| **CLAUDE.md** | 給 Claude 參考的專案／個人指南，屬於建議而非強制 |
| **Skills** | 按需載入的可重複使用流程，平常不佔用上下文 |
| **Subagents** | 有獨立上下文與工具權限的子代理，處理特定子任務 |
| **MCP** | 連接外部工具或資料來源（資料庫、瀏覽器自動化等） |
| **Hooks** | 在特定事件（如工具呼叫前後）強制觸發自動化指令 |

---

## 設定檔的階層架構

Claude Code 的設定並非只有一份檔案，而是依照「範圍」分成好幾層，載入時會全部合併：

| 範圍 | 檔案位置 | 用途 | 是否納入版控 |
| --- | --- | --- | --- |
| **Managed（企業）** | macOS：`/Library/Application Support/ClaudeCode/managed-settings.json`<br>Linux/WSL：`/etc/claude-code/managed-settings.json`<br>Windows：`C:\Program Files\ClaudeCode\managed-settings.json` | IT／DevOps 統一管理的組織政策 | 由企業統一部署 |
| **User（使用者全域）** | `~/.claude/settings.json` | 個人在所有專案通用的偏好設定 | 否 |
| **Project（專案共用）** | `.claude/settings.json` | 團隊共用的專案設定 | 是 |
| **Local（專案個人）** | `.claude/settings.local.json` | 個人在該專案的私人設定 | 否（建議加入 `.gitignore`） |

**優先順序（由高到低）：** Managed > 命令列引數 > Local > Project > User。也就是說同一個欄位若多層都有設定，會採用範圍較窄、較貼近當下情境的那一層；不過 `permissions` 規則比較特別，是**跨層合併**的，而且評估順序是 deny 優先於 ask，ask 又優先於 allow。

設定檔修改後大多數欄位會即時生效，不需要重開 Claude Code；可以用 `/status` 查看目前實際套用了哪些設定來源，或用 `/doctor` 檢查設定檔是否有誤。

---

## permissions：權限規則怎麼寫

`permissions` 是 `settings.json` 裡最常用到的欄位，用來控制 Claude 可以「不問就做」、「一定要問」還是「完全不能做」哪些操作。規則的語法是 `工具名稱(比對樣式)`，樣式支援 `*`（單層萬用字元）與 `**`（多層萬用字元）。

以下是我自己 `~/.claude/settings.json` 裡實際在用的一小段設定：

```json
{
  "permissions": {
    "allow": [
      "Bash(npm install *)",
      "Bash(npm run *)",
      "Skill(run)",
      "Skill(run:*)",
      "Bash(chromium-cli --help)"
    ]
  },
  "effortLevel": "xhigh"
}
```

再搭配一個常見的 `deny` 範例，避免誤觸敏感操作：

```json
{
  "permissions": {
    "allow": [
      "Bash(npm run lint)",
      "Bash(npm run test *)"
    ],
    "deny": [
      "Bash(curl *)",
      "Read(./.env)",
      "Read(./secrets/**)"
    ],
    "ask": []
  }
}
```

常見可搭配的工具前綴包含 `Bash(...)`、`Read(...)`、`Write(...)`、`Edit(...)`，以及 MCP 工具的 `mcp__<server>__<tool>` 格式。

---

## 常見設定欄位

除了 `permissions`，`settings.json` 還能設定不少行為，例如：

```json
{
  "model": "claude-opus-5",
  "effortLevel": "xhigh",
  "language": "traditional chinese (taiwan)",
  "env": {
    "DEBUG": "1"
  },
  "autoMemoryEnabled": true,
  "outputStyle": "tui"
}
```

* `model`：指定要使用的模型。
* `effortLevel`：控制推理強度（如 `low`、`medium`、`high`、`xhigh`）。
* `language`：指定 Claude 回覆時使用的語言（就是這篇文章開頭範例裡我自己設成 `traditional chinese (taiwan)` 的欄位）。
* `env`：注入環境變數給每次工具呼叫使用。
* `autoMemoryEnabled`：是否啟用下面會介紹的「自動記憶」功能。
* `outputStyle`：調整輸出風格。

這類欄位相當多，實務上不需要一次記住，遇到需求再查官方文件對照即可。

---

## CLAUDE.md：給 Claude 的持久指南

如果說 `settings.json` 是「技術上的強制規則」，`CLAUDE.md` 就是「給 Claude 參考的上下文說明」，每次對話開始時都會自動載入，但**沒有強制力**，Claude 是把它當作建議來遵循。

放置位置一樣分好幾層，載入順序由「範圍最廣」到「範圍最窄」：

| 範圍 | 位置 | 用途 |
| --- | --- | --- |
| Managed（企業） | 同 `managed-settings.json` 所在資料夾下的 `CLAUDE.md` | 全組織一致的規範（如程式碼規範、合規要求） |
| User（個人全域） | `~/.claude/CLAUDE.md` | 個人在所有專案都適用的偏好 |
| Project（專案共用） | `./CLAUDE.md` 或 `./.claude/CLAUDE.md` | 團隊共用的專案架構、慣例、指令說明 |
| Local（專案個人） | `./CLAUDE.local.md` | 個人在該專案的私人筆記（建議加入 `.gitignore`） |

幾個實用細節：

* 可以直接下 `/init`，Claude 會分析目前的程式碼庫，自動產生一份包含建置指令、測試方式、專案慣例的 `CLAUDE.md` 草稿。
* 官方建議單一 `CLAUDE.md` 控制在 **200 行以內**，內容太長會稀釋上下文、降低遵循度；篇幅大的專案可以拆到 `.claude/rules/` 目錄下，每個檔案負責一個主題，甚至能用 YAML frontmatter 的 `paths` 欄位限定只在特定檔案類型才載入。
* 支援 `@path/to/file.md` 語法匯入其他檔案內容。

---

## Auto Memory：AI 自動累積的記憶

除了人工寫的 `CLAUDE.md`，Claude Code 還有一套「自動記憶」機制：Claude 會依照對話中觀察到的修正、偏好、除錯心得，主動寫成筆記存起來，下次同一個專案的對話就能沿用，不用每次重新解釋一遍。

* 存放位置：`~/.claude/projects/<project>/memory/`，同一個 git repo 底下所有 worktree 共用同一份記憶。
* 進入點是 `MEMORY.md`，每次對話開始只會載入前 200 行（或 25KB，以先到者為準），細節則拆到其他主題檔案，由 Claude 依需要自行讀取。
* 可以用 `/memory` 指令瀏覽、編輯或刪除這些記憶檔案，內容都是純 Markdown。
* 若不想要這個功能，可在 `settings.json` 設定 `"autoMemoryEnabled": false` 關閉。

這個功能有點像是 Claude 幫自己維護一份「跟你合作的心得筆記」，跟 `CLAUDE.md` 互補：`CLAUDE.md` 是你主動告訴 Claude 的規則，Auto Memory 則是 Claude 自己觀察後歸納的內容。

---

## Hooks：在特定事件自動觸發指令

`hooks` 讓你能在 Claude Code 生命週期中的特定時機，強制執行一段 shell 指令（或 HTTP 請求），且**不受 Claude 決策影響**，是真正的強制執行層。比較常用到的事件包括：

* `SessionStart`：對話開始或恢復時。
* `PreToolUse` / `PostToolUse`：工具呼叫前／後。
* `UserPromptSubmit`：使用者送出提示前。
* `Stop`：Claude 完成一次回應時。
* `Notification`：Claude Code 發出通知時。

（實際支援的事件遠不只這些，完整清單可查官方文件。）

一個常見用法，是在每次編輯或寫入檔案後自動跑格式化工具：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "npx prettier --write \"$CLAUDE_FILE_PATH\""
          }
        ]
      }
    ]
  }
}
```

`matcher` 用來篩選要觸發的工具名稱，`hooks` 陣列則是實際要執行的動作，`type` 除了 `command`（執行本機指令）之外，也支援 `http`、`mcp_tool`、`prompt` 等類型。

---

## .mcp.json：擴充外部工具（MCP Servers）

**MCP（Model Context Protocol）** 讓 Claude Code 能連接外部工具或資料來源，例如資料庫、瀏覽器自動化、第三方 API 等。專案層級的設定放在專案根目錄的 `.mcp.json`，方便整個團隊共用：

```json
{
  "mcpServers": {
    "playwright": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@playwright/mcp@latest"]
    }
  }
}
```

個人層級的 MCP 設定則會存在 `~/.claude.json` 裡，不會與團隊共用。

---

## 自訂 Slash Commands 與 Skills

Skills 是一種「按需載入」的可重複使用流程，平常不會佔用上下文，只有在符合觸發條件、或被明確呼叫時才會載入進對話。放置位置為 `.claude/skills/<skill-name>/SKILL.md`（專案層級）或 `~/.claude/skills/<skill-name>/SKILL.md`（個人全域），本質上就是一份帶有 YAML frontmatter 的 Markdown 說明文件，內容通常包含 `name`、`description` 等欄位，以及實際的操作步驟。

這類機制很適合拿來封裝「重複性高但不需要每次都在上下文裡」的工作流程，例如專案的部署流程、特定的程式碼審查清單等。

---

## 自訂 Subagents

Subagent 是擁有獨立上下文、獨立系統提示詞與獨立工具權限的子代理，適合用來處理「會產生大量中間結果、但主對話不需要記住細節」的工作，例如搜尋程式碼、整理研究資料等。設定檔放在 `.claude/agents/<name>.md`（專案層級）或 `~/.claude/agents/<name>.md`（個人全域），格式同樣是 YAML frontmatter + Markdown：

```markdown
---
name: code-improver
description: Scans files and suggests improvements for readability, performance, and best practices. Use after writing or modifying code.
tools: Read, Grep, Glob
model: sonnet
---

You are a code improvement specialist. For each issue you find, explain
the problem, show the current code, and provide an improved version.
```

* `name`：subagent 名稱，也是之後呼叫它的依據。
* `description`：描述用途，Claude 會依此判斷什麼時機該委派給這個 subagent。
* `tools`：限定這個 subagent 能使用哪些工具，達到「權限最小化」的效果。
* `model`：可以指定較便宜的模型（例如 Haiku）來處理單純的子任務，控制成本。

---

## 小結

整理下來可以發現，Claude Code 的設定機制大致分成兩大類：

* **強制執行層**：`settings.json`（含 `permissions`）與 `hooks`，不論 Claude 怎麼判斷都會被套用。
* **上下文參考層**：`CLAUDE.md`、Auto Memory、Skills，屬於「建議」而非「強制」，靠 Claude 自行理解與遵循。

再加上 `.mcp.json` 擴充外部工具、`agents/` 拆分子任務，整體組合起來就能把 Claude Code 客製化成很貼近自己（或團隊）工作習慣的開發助手。建議可以先從最基本的 `permissions` 與 `CLAUDE.md` 開始調整，等熟悉之後再視需求加上 hooks、subagents 等進階設定。

---

### 參考來源

- [Claude Code 官方文件 - Settings](https://code.claude.com/docs/en/settings)
- [Claude Code 官方文件 - Memory（CLAUDE.md / Auto Memory）](https://code.claude.com/docs/en/memory)
- [Claude Code 官方文件 - Hooks](https://code.claude.com/docs/en/hooks)
- [Claude Code 官方文件 - Subagents](https://code.claude.com/docs/en/sub-agents)
