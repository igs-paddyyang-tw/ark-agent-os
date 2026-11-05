# Changelog — ark-agent-os

## 2.1.1 (2026-11-05)

> Kiro 月額度用完（11-05 實際發生）時，整隊一行指令切到 claude、恢復後切回。規格 paddy-bot `docs/specs/backend-failover-switch-spec.md`（依 ninja 當天原型）。

- **`team.yaml backend_sets:`**（選填）：「主要／備援」各組的 backend＋model，`overrides` 逐 instance、`exclude` 排除；`active_backend_set` 記目前在哪組、`backend_set_pins` 個別 instance 釘在某組。**沒設＝行為完全不變**
- **`ark-agent-os backend status [--json]`**：每個 instance 在哪組、backend／model、執行狀態，被組覆寫的標出原值
- **`ark-agent-os backend switch --to <組> [--only a,b] [--dry-run]`**：只改 `active_backend_set`／`backend_set_pins` 兩行（其餘一字不動，回切後逐位元組相同）→ reload → **只重啟執行中且有變的** instance、逐一等就緒；起不來的自動釘回原組。claude-code 的 model 不可為 auto、找不到 claude-agent-acp → 切換前就擋
- **額度告警內建**：daemon 偵測到額度類重複失敗（含啟動時）→ 私訊管理者，附可直接貼上的切換指令（`failover.notify_cooldown_hours` 內不重發）；`failover.mode: auto` 才自動切，並每天 00:10 嘗試切回
- `/api/health.backend_set`；在備援組時 degraded 標示（附回切指令）

## 2.1.0 (2026-11-05)

> TG 系統訊息：**看得出是哪一隊**＋**文字可由設定檔改**（設計 paddy-bot `docs/designs/tg-message-templates-design.md`）。

### 🆔 團隊識別

- `team.yaml` 新增 `identity:`（name／emoji／title／env，全部選填）。沒寫就推導：隊名 deploy.yaml name → home 目錄名、emoji 取 manager 代號、環境取 `.env` 的 `ARK_ENV_TYPE`（aipc→AIPC、cloud-prod→雲端、pc→本機）
- `/start`、`/status`、`/help`、啟動／重啟橫幅、告警（hang、成本、速率限制、崩潰、記憶體）開頭帶「🐟 fish-team｜AIPC」；對話中的回覆不加

### 🗂️ 訊息目錄（約 260 則）

- 所有送 TG 給人看的系統訊息搬進 `ark_agent_os/texts/zh-TW/*.yaml`，每則一個 ID；**預設文字逐字不變**（快照比對），刻意改版的只有下列畫面
- 部署覆寫：home 的 `messages.yaml`，或 `team.yaml messages.overrides`。覆寫錯 ID／錯占位符 → 啟動時列出並附建議值；執行期渲染失敗退回套件預設，訊息照送
- **once 機制**：`messages.file_policy: once`（預設）→ home 沒有 `messages.yaml` 時寫一份**全部註解**的範本（每則的預設文字＋可用占位符），之後永不覆寫；因為不寫入實際值，升版後新的預設文字照樣生效。不想要這個檔：`file_policy: skip`
- CLI：`ark-agent-os messages list｜diff｜template`
- 守門：送給人的呼叫裡出現寫死的中文／emoji 字面值 → 測試紅

### 🖼️ 改版的畫面

- `/status`：「存活 x/y」改「常駐 x/y · 待命 n 💤 · 異常 n」—— 延後啟動的成員不再算成故障；異常直接列前 3 項；⚪ 只留給真的停掉的成員並附圖例
- `/start`：團隊識別＋title、「N 位 agent（常駐 n）」
- 啟動橫幅：各層只列常駐中的成員、待命另列一行（原本列兩次）、預算／時間／版本收成兩行、版本寫 ark-agent-os（原寫舊名 ark-team-agent）
- 代號推導：description 沒有「代號 — 職稱」分隔時改用 `display_name`，都沒有用 instance 短名（原本整段 description 當代號，nana-bot 橫幅出現長句）

### 🐛 盤點順帶修正

- crash 冷卻／記憶體過高不再顯示成「閒置 0 分鐘」（各自一則）
- 閒置告警按鈕原標「⏸ 暫停 X 今日」，實際動作是請它向 leader 要工作 → 標籤改成實際動作
- 成本暫停訊息的時區改讀 `cost_guard.timezone`（原寫死 Asia/Taipei）
- 轉呈逾時訊息的「管家」改成實際 manager 名稱
- 組 topic 送出失敗不再把例外原文送進群組（只進 log）
- 每日報告時間可設：`notifications.daily_summary_time`（預設 21:00）
- 原本 HTML 訊息裡沒 escape 的值（代號、錯誤訊息、L3 問題）現在會 escape —— 值含 `<` 時不再讓 TG 回 400
- `/api/health` 版號：editable 開發源原本回報 team 的 1.13.23，改回報實際的 os 版號
- `deploy doctor／init` 認得 `-m ark_agent_os` 的 unit（nana-bot 升級驗證誤報「exec 描述 module／實際 script」）
- 背景任務終態回報（獨立子程序）也套用部署的覆寫與識別

### 🔄 相容性

- 沒寫 `identity`／`messages` 的部署照常啟動；首次啟動在 home 多一個全註解的 `messages.yaml`（無作用）
- 回滾到 2.0.x：新的兩個頂層鍵會被忽略（2.0.3 實測），可直接降版

## 2.0.4 (2026-11-04)

> nana-bot 11-04 以 `migrate --from-bot` 遷到 2.0.2、與 nana 子 team 預覽 `deploy init` 時踩到的套件 bug。

- **🔴 migrate 產出 `backend: auto` 整隊跑不起來**：bot `backend.cli: auto` 原樣寫出，引擎不認 → 全部 CONFIG_INVALID；migrate 自己的驗證卻判通過。改成 `auto`／`kiro` → `kiro-cli`，驗證改走與 daemon 相同的 backend 解析
- **內建網站 `/wiki` 500**：wiki 頁 `tags` 有 YAML 整數（`2025`）時與字串混排拋 TypeError。建索引時 tag 一律轉字串（wiki 禁手改，只能套件端處理）
- **leader 延後啟動時每次重啟都發「📋 新的一天」**：原本查 leader 的 START 事件判斷重啟，但延後啟動的 leader 在判斷之後才寫入；查詢也沒篩日期（常駐 leader 隔天起永遠判成重啟）。改看 daemon 自己的「今天啟動次數」
- **遷移後幾乎每則回覆都附 HTML**：bot 的 `report.min_chars: 150` 照搬，但 2.0 是「同一輪累計字數 ≥ 門檻」→ 700 字的回覆也附檔、內容與文字重複。migrate 遇低於 3500 的門檻改為 3500 並註記
- **補送回覆的報告附件晚 2 分鐘才到**：FR-038 補送在 turn 結束後才發，新開的報告緩衝等 120 秒閒置。一輪已結束時改走 5 秒收尾寬限
- **deploy 對子 team（共用上層 venv）判斷全錯**：`deploy init` 依服務 ExecStart 推 venv（`../../.venv`）、找得到停用中的服務、找不到服務時不標啟用、套件名依版號；`upgrade` 遇共用 venv 擋下、`setup` 不另建 venv 也不裝套件

## 2.0.3 (2026-11-04)

- **manager 派工的「已交給」多列了被擋下的成員**：nana 實測回「已交給 羅賓、娜美」—— manager 先試著直接派 worker（羅賓）被權限擋下（403），再改派 leader（娜美）。改成**送達才記**，只列真正交辦的對象

## 2.0.2 (2026-11-04)

> 2.0.1 上線後 nana／paddy TG 實測的兩個體驗問題。

### 🧹 補送不再夾帶工作旁白

- **現象**：agent 沒呼叫 `reply()` 時整輪輸出補送給使用者，開頭夾著「有足夠的設計文件…讓我再看一下 team.py」這類工作旁白；長答覆報告的標題是使用者的提問句
- **修正**：補送只取**最後一個工具呼叫之後**的文字（那段是空的就退回整輪）；句尾標點後直接黏著 Markdown 標題的短旁白一併去掉。報告正文第一行是 H1 → 用它當標題

### 📮 manager 派工當下先回一句

- **現象**：使用者私訊 manager，manager 轉派後不說話（轉派會清掉補送），結果回來前約 1 分鐘看不到任何回應
- **修正**：manager 在「使用者等回覆」的一輪裡轉派、且該輪沒呼叫 `reply()` → turn 結束時以 manager 身分補一句「📮 已交給 {成員} 處理，完成後回報給你」。只對私訊來源；manager 自己回過就不補

## 2.0.1 (2026-11-04)

### 🔴 kiro-cli 3.0 相容：kiro backend 一律走 ACP

- **現象**：kiro-cli 2026-11-04 11:51 自動升到 3.0.0，移除了 v1 引擎；走 chat transport（`--legacy-ui --agent-engine=v1`）的 worker 啟動全部逾時（ninja ai-dev 連 3 次）。ACP 不受影響
- **修正**：偵測到 kiro-cli ≥ 3 時，kiro backend 一律走 ACP —— **不論 profile 寫 `transport: chat` 或 `ARK_KIRO_ACP_ENABLED=0`**（log 寫明原因）；版本只偵測一次，偵測失敗維持既有行為
### 🧹 build 殘留的 `.egg-info` 不再被當成「已安裝」

- **現象**：build wheel 會在 `src/` 留下 `<pkg>.egg-info`；editable 開發源（src 在 sys.path）裡 importlib.metadata 把它當成已安裝 →
  混裝偵測誤判「ark-agent-os＋ark-team-agent」、版號讀成 2.0.0。開發源只要 build 過 os 就會**拒絕啟動**
- **修正**：版號與混裝偵測只認真正安裝的 `.dist-info`

- 部署不必改設定；team 1.13.23 已凍結不修 → 仍在 team 上、有 kiro chat agent 的部署請升 2.0.1，或改 `backend: kiro-acp`（ninja 已改，5034b15）

## 2.0.0 (2026-11-04)

> **ark-agent-os 第一版 = ark-team-agent 最後一版（1.13.23）**，兩者由同一個 commit build。
> `ark_team_agent` 與 `ark_bot_agent` 併入單一套件 `ark_agent_os`：一個 daemon，以 `team.yaml` 的 profile 區分 bot（單一 manager）與 team（manager＋admin＋leader＋workers）。

### 🧭 架構

- **一個 daemon**：原 ark_team_agent 引擎（instance 管理、lazy spawn、熔斷、cost_guard、scheduler、TG、API）＋ ark_bot_agent 的獨有能力（`/job` 背景任務、長答覆報告、上傳、記憶 recall／chat trace、兩層存取、`/publish`）
- **根目錄＝manager**（使用者的分身，私訊入口）；`agents/` 是團隊（admin 維運、leader 派工驗收、workers 分工）—— 運作準則見 paddy-bot `docs/reports/2026-11-03-ark-agent-os-operating-principles.md`
- **相容 shim**：`import ark_team_agent`、`from ark_bot_agent import run_bot` 照常可用（帶 bot 專屬參數或沒有 team.yaml → 指路 `migrate`），M5 移除
- CLI：`ark-agent-os`（`ark-team-agent` 為別名）

### 🆕 相對 ark-team-agent 1.13.x 的 2.0 預設

| 設定 | 2.0（ark-agent-os） | team 1.13.x |
|---|---|---|
| `communication.chat_receiver_policy` | **block**（收訊息的 agent 走 kiro chat transport → 拒絕啟動，附 kiro-acp 改法） | warn |
| `init --profile` 新部署 | `reply_to_origin_private: true`、`relay_via_manager: true` | — |
| manager 工作目錄 | 專案根 `.`（`init`、`migrate` 預設） | — |

### 🔄 從 ark-team-agent 1.13.x 升級

```bash
ark-team-agent deploy upgrade --to 2.0.0     # 在 1.13.23 上執行：先移除 ark-team-agent、再裝 ark-agent-os，失敗自動裝回
```

- `deploy.yaml` 的 `package.name` 會一併改為 `ark-agent-os`
- 同一 venv 同時裝兩個發行版會拒絕啟動（混裝偵測，訊息附修正指令）
- bot 部署：`ark-agent-os migrate --from-bot <目錄>`（dry-run 預設；manager 留在原目錄；bot 專屬設定列出不對應項）

### ✅ 2.0 期間完成的可靠性修正（team 1.13.14～1.13.23 已先行出貨）

- 服務重啟不遺失在途訊息（停機 drain＋overflow 持久化重送，KillMode=mixed）
- ACP 一次一輪：忙碌時的訊息不再被 `Prompt already in progress` 吃掉
- 派工鏈結果經 manager 轉呈（`relay_via_manager`）；worker 回報交 leader 驗收
- 全來源未呼叫 reply 補送、外洩過濾、長答覆同一輪累計附件、直達成員寫進 manager daily
