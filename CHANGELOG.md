# Changelog — ark-agent-os

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
