# Changelog — ark-agent-os

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
