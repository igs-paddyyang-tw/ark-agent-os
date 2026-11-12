# Changelog — ark-agent-os

## 2.1.18 (2026-11-12)

- **用量守門 `usage_guard`：同時算成本（USD）與呼叫次數，兩者皆可告警與暫停**（kiro-bot 雲端維運 11-10 提；問題單 `docs/issues/2026-11-10-usage-guard-cost-and-requests.md`；規格 `docs/specs/issues-1110-batch-spec.md` FR-006～013）。原本只靠 agent 自報 USD，雲端三隊整天 $0，額度用完時全隊靜默
  - 設定頂層改 `usage_guard`（`cost_guard` 仍可用，並存時 `usage_guard` 優先並記 warning）；巢狀 `cost`／`requests`（`daily_limit`、`warn_at_percentage`、`action_on_limit: none|warn_only|pause`、`per_instance_limits`、`count_failed` 預設 true）／`exempt`／`notify`。**舊設定不改行為不變**
  - daemon 自動計次：**真正送出 prompt 時**才 +1（`_queue_worker` 兩處投遞；存 overflow／過期／取消不計、重播計 1 次），`send_input` 例外另記 `failed_requests`；入口擋下記 `rejected`
  - 達限且 `pause`：新訊息不入隊；**使用者會收到固定提示**（TG 私訊、指令路由、web `/api/chat/send` 帶 `notice`），不再靜默或誤說「離線」；`exempt` 照計次但不被擋；進行中的一輪跑完不強殺
  - 各維度各自判斷 `action_on_limit`；上限為 0 的維度＝不限制（當天殘留的暫停旗標不再生效）
  - 告警（warn 80%／達限／恢復）走 `DaemonAPI.notify_paddy`，同事件每日一次；日界線自動恢復，`POST /api/costs/resume` 手動恢復
  - `/api/health.usage`、`/api/costs` 加次數欄位；日報附當日次數
  - SQLite `spends` 加欄位（冪等 migration，舊資料為 0）
  - 守門 7 支（設定、CostGuard、閘門、API、相容反證、入口出口 AST、使用者提示）；paddy 實機：web 與 TG 私訊達限各只回一則提示、owner 告警送達、上限調回 0 後閘門與 health 一致放行
  - ⚠ 未做：團隊總量上限；web 前端是否顯示 `notice` 未驗證；次數只是 Kiro credit 的代理指標

## 2.1.17 (2026-11-10)

- **preflight `exit_liveness` 把 P2P 政策拒絕當送出失敗**（雲端 game-team 11-10：worker 間 P2P 超輪 403 共 27 次，重啟後 3 個 P0 誤報；問題單 `docs/issues/2026-11-10-preflight-exit-liveness-p2p-policy-false-p0.md`）：`tool_calls.log` 新增 `✗policy`（`/api/send` 403、`allowed_targets` 拒絕），`exit_liveness` 不計它；真失敗（非 403、連線錯誤）仍 `✗`→P0；舊 log 的 `✗` 照舊視為真失敗
  - 工具回傳改「發送被擋（政策）／委派被擋（政策）: <daemon 的說明>」—— agent 直接看到升級路徑，不再是一整包 dict
  - 新 P2 探針 `p2p_blocked_rate`：1 小時內被擋 ≥ 5 次 → 提示 worker 間討論過多（每輪都是一次 LLM 回合）；P2 不進 `degraded`、不單獨通報
  - 守門 `TestPolicyRejection`（AC-1～4）、`TestPolicyMark`；反證：拿掉 preflight／team_mcp 的分類各自紅
  - ⚠ 升級前已寫入的舊 `✗`（實為政策拒絕）仍視為真失敗 —— 誤報殘留到該 instance 下一次成功送出，或滿 24 小時判陳舊才消失；`POST /api/preflight/run` 清不掉

## 2.1.16 (2026-11-10)

- **secret_scan 誤報**（AIPC 11-10 回報；問題單 `docs/issues/2026-11-10-secret-scan-false-positives.md`）：`sk` 規則加左邊界（`task-closeout`、`risk-tier` 尾端的 `sk-` 不再命中）；`assign` 值不以 `` $ ` \ ( * < `` 開頭（`TOKEN=$(grep …)`、`token = <your-token>`、`***masked***` 不再命中）。真樣式（`sk-…`、`sk-ant-…`、`ghp_…`、`password=…`）仍命中
  - `secret_audit` 的 P0 訊息講明 ack 檔在**專案根** `state/secret_audit_ack.json`（寫到 `agents/<x>/state/` 不會生效）
  - `redaction-vectors.yaml` 補 6 負例、3 正例；新守門 `tests/test_secret_scan_false_positives.py` 直接對 `ark_agent_os.secret_scan` 跑全部向量（既有 parity 測試 27 條跳過，不能當證據）。反證：拿掉左邊界、拿掉 assign 排除、拿掉 P0 提示各自紅

- **`/publish` 確認卡與套件預設改成實際行為**（Paddy 11-10）：卡片原寫「管理 agent 會更新 MEMORY.md / README.md」—— 本機部署根目錄都沒有 MEMORY.md（記憶導覽在 `.kiro/steering/`），team 版也不派 agent 改寫。改列實際發布的 `README.md`、`TODO.md`（或 `docs/TODO.md`）、`memory/daily/`
  - 白名單加 `TODO.md`、`docs/TODO.md`；新增 `PUBLISH_DEFAULT_FILES`（/publish 預設帶的檔，不含 MEMORY.md；MEMORY.md 仍在白名單只為相容直接呼叫 `publish_repo` 的舊呼叫端）
  - 預設 commit 訊息單一來源：確認卡原顯示「由管理 agent 產生」、實際 commit 卻用另一句寫死字串 → 改成 `cmd.publish.default_message`＝實際 commit 訊息（`chore: 發布收尾更新（memory/daily + readme + todo）`）
  - 守門：確認卡列的檔＝`PUBLISH_DEFAULT_FILES`、不含 MEMORY.md；預設訊息兩處同源（反證各自紅）
  - 部署端 `messages.yaml` 是全註解範本、無覆寫 → 升版即生效；範本註解裡的舊文字不影響行為
- **failover 額度告警送不出去**（hoyeah 2.1.15 實錄：`'Daemon' object has no attribute 'notify_paddy'`）：`FailoverGuard` 綁的是 `Daemon`，`notify_paddy` 只在 `DaemonAPI` → 每次額度用完的 TG 告警都失敗、只留 log。改在 `run_team` 建好 DaemonAPI 後改綁（比照 decision_mgr）。原測試用自帶 `notify_paddy` 的假 daemon 所以一直綠 → 新守門檢查真實類別形狀＋實際綁定（反證紅）。問題單 `docs/issues/2026-11-10-failover-notify-paddy-wrong-object.md`

## 2.1.15 (2026-11-09)

> 原本要以 2.1.14 發（hotfix），但網站帳號已先以 2.1.14 發布（我方未先在遠端佔號而撞號）→ 改發 2.1.15，以 main 為基底（含 2.1.14 帳號功能，預設關閉）

- **TG `/status` 每位成員附設定**（Paddy 11-09：「不要只有現在的狀態，要附上設定狀態像模型資訊」）：改成每層一個標題、成員逐行 —— `狀態圖示 代號　後端·模型（·推理強度）　啟動方式`，例 `🟢 👨‍💻 Paddy　claude·opus　常駐`、`💤 赤木　claude·sonnet　待機·閒置 30 分關`。原本每層成員擠一行只有狀態圖示，待命成員只剩名字清單
  - 模型與後端用 `get_status` 的實際值（backend_sets 套用後）；沒在跑的成員退回設定值；後端名依前綴縮寫（`claude-code`／`claude-acp` → claude、`kiro-cli`／`kiro-acp` → kiro）
  - 啟動方式：常駐／待機（`auto_start: false`）／待機·閒置 N 分關／開機起·閒置 N 分關／開機起（`persistent: false`＋不回收）
  - 不在 primary 後端組時：頂部加「🔀 後端組 X（回切指令）」，被覆寫的成員標 `↪X`；在 primary 組不標（primary 就是正式設定）
- 啟動橫幅不變（仍為精簡版）；訊息目錄新增 `cmd.status.layer／agent／mode_*／model_default／mark_set／backend_set`，移除 `cmd.status.group／group_cont／standby`（無部署覆寫）

## 2.1.14 (2026-11-09)

> **網站帳號**：信箱＋密碼註冊登入、Google 登入、三級角色（viewer／operator／owner）、owner 存取權限審批頁。
> 移植自 nana 先行版（2026-11-06 上線）；計畫 `docs/plans/ark-os-web-accounts-port-plan.md`，規格／設計 `ark-os-google-login-*`。
> 版號：Paddy 11-09 拍板沿 2.1.x、不跳 minor（原訂 2.2.0 作廢）。
> **預設關閉**：沒設 `AUTH_ACCOUNTS`／`web.auth.accounts` 的部署，登入行為與 2.1.x 完全相同。

- **開啟方式**：`.env` 設 `AUTH_ACCOUNTS=1`、`AUTH_OWNERS=<你的信箱>`（或 team.yaml `web.auth.accounts／owners`），重啟後用該信箱註冊即成為 owner。使用指南 `docs/guides/website-accounts.md`
- **註冊申請 → owner 審批**：未列名者註冊後進等待頁，owner 在後台「存取權限」（`/app/access`）核准並指定角色，申請人**自動登入**（簽章 cookie，不必再輸密碼）；新申請經 daemon 的 TG 通知 owner
- **三級角色**：viewer 只看；operator 可對話、上傳、操作；owner 另可管權限、系統、排程。每個請求重讀帳號 —— 停用、改角色下一個請求生效
- **安全**：密碼 scrypt、5 次失敗鎖 15 分鐘；所有寫入 CSRF double-submit（頁面自動帶 header）；公開唯讀（`CONSOLE_PUBLIC_READ`）不含 owner 頁；最後一位 owner 不能降權；team.yaml `web.auth.users` 鎖定名單網頁不可改
- **Google 登入**（選配）：`pip install 'ark-agent-os[google]'`＋`GOOGLE_CLIENT_ID／SECRET`；OIDC 授權碼＋PKCE、nonce、信箱已驗證、帳號 sub 綁定。redirect URI 非 localhost 需 HTTPS 正式網域
- **CLI** `ark-agent-os auth doctor／users／approve／deny／role／disable／enable／revoke／break-glass`：break-glass 產生 15 分鐘、單次的緊急登入連結，使用時通知所有 owner
- **health** 新增 `auth`（mode、帳號數、待審數、owner 數、30 天 break-glass 次數；不含個資）
- **自建網站**：`from ark_agent_os.website.auth import mount_auth`；同名頁面範本優先用自建網站的，只掛 auth 路由
- session 表加 `email` 欄（舊庫自動遷移）；系統頁版本卡改顯示 ark-agent-os
- 修 `test_web_migrate_dryrun_runs`：改用暫存 home（9a3c860 封存 apps/team-website 後失效）
- 測試：帳號 45 項（Phase 1 18、Phase 2 16、Phase 3 11），反證 11 項各自紅

## 2.1.13 (2026-11-09)

> 由 v2.1.12 tag 開 hotfix 分支發佈：main 上另一個 session 的網站帳號功能（Phase 0+1，計畫 2.2.0）**不在本版**。依據：`docs/reports/review/2026-11-09-ark-agent-os-architecture-review.md` F-3、F-7

- **instance 沒註冊時講清楚原因**（F-3）：`auto_start: false`＋`persistent: true` 原本派工只報 `not defined in team.yaml`（明明有定義）→ 改報「這是本體工作區語意、要首次派工才啟動請改 `persistent: false`」，**開機時就警告**，不等派工失敗（2026-11-06 paddy 宮城 40 秒派不到）
- **agent 自我重啟防護**（F-3）：systemd 底下，daemon 在 agent 子程序 PATH 前置 `systemctl` 包裝（`state/agent-bin/systemctl`）—— `restart` 自己的服務改送 SIGTERM 給 daemon（systemd `Restart=always` 拉起）；`stop`／`kill` 自己的服務拒絕；其他指令原樣交給真的 systemctl。原本 agent 自我重啟會在停機時殺掉自己的 systemctl，服務停在 inactive（2026-11-06 wukong 31 分鐘）
- **`POST /api/preflight/run`**（F-7）：重跑 preflight 並更新 `degraded`，不必重啟；secret_audit 的提示改成「ack 查證過的命中後呼叫此 API」（原本 ack 完 health 仍紅，只能重啟，2026-11-09 kiro-bot）

## 2.1.12 (2026-11-09)

- **reply_file 的 TG 上傳錯誤不再變成 500**（11-09 paddy ai-dev 新聞日報：`httpx.ReadTimeout` 直接拋成 ASGI exception，agent 只看到呼叫失敗、可能重送）：
  - 送檔（document／photo）改帶 `FILE_SEND_TIMEOUTS`（read／write 60 秒；PTB 預設 read 只有 5 秒，上傳 HTML 報告常在 TG 處理中就逾時）
  - API 統一攔截：逾時回 `{"ok": false, "maybe_delivered": true}` 並明說「可能已送達、別立刻重送」；其他 TG 網路錯誤回 `ok: false`＋原因；白名單外副檔名回訊息（不再 500）
  - 守門：掃全套件，每個 `bot.send_document／send_photo` 都要帶 `FILE_SEND_TIMEOUTS`（掃到少於 4 處就紅，防範圍掃錯）

## 2.1.11 (2026-11-06)

> 原本標 2.1.10，但另一個 session 已先以 2.1.10 發布（辦公室 hang／error 亮燈）→ 改發 2.1.11，內容不變


- **修 TG 回覆後備出口的 `<b>` 被跳脫**（2.1.2～2.1.9）：agent 沒有 topic／私訊出口時（例如 paddy 的 admin），reply 改送 admin 私訊並加代號前綴（`dispatch.reply_no_exit`）—— 這條把整段內文 `_html_escape`，「⏰ <b>狀態更新</b>」照樣顯示字面標籤（11-06 17:00 paddy 赤木）。改走 `_md_to_html`（escape-first＋只還原成對 b／i／u／s／code）。2.1.2 修內文、2.1.7 修標題行，這條兩次都漏
- 守門 `test_no_tg_exit_escapes_agent_text_wholesale`：掃整個 `api.py`，agent 文字（text／req.text／body）傳進 TG 範本不得用 `_html_escape`；並檢查 `_md_to_html` 出口數量，掃到 0 就紅（防「範圍掃錯」）

## 2.1.10 (2026-11-06)

- **辦公室：hang／error 也會亮 error**（nana 回饋，`docs/issues/2026-11-06-pixel-office-p1-review-vs-2.1.8.md` R-4 補遺）：2.1.9 只有 status 變 crashed 才亮，hang 偵測、預算超限、設定錯誤、重複失敗等只寫 event_log 的錯誤，辦公室看不到。新增 `event_log.add_listener`（`record()` 寫入後、在事件迴圈上呼叫；listener 例外吞掉），daemon 登記 `ActivityHub.on_event`：hang／error／crash → `last_error`（error 顯示 30 秒）。未知 instance（排程、系統）不建假 agent
- 測試：`tests/ark_agent_os/test_activity_error_events.py` 6 項；反證：拿掉 listener 通知／daemon 登記／instance 檢查各自紅

## 2.1.9 (2026-11-06)

- **Pixel Office P1 審查修正**（nana 回饋 `docs/issues/2026-11-06-pixel-office-p1-review-vs-2.1.8.md`；設計 `docs/designs/acp-tool-events-design.md` §7）：
  - **R-3 狀態不回落**：TTL 回落原本只在 SSE 佇列閒置 15 秒時 tick → 只要有一個 agent 持續出事件，其他 agent 的 writing／thinking／error 永遠不回到 idle。改成 ActivityHub 自帶 ticker（每 2 秒；第一個訂閱者啟動、沒人訂閱自動結束）
  - **R-4 回覆中／思考／context／權限從沒出現過**：`process_acp` 把 `agent_message_chunk`→writing、`agent_thought_chunk`→thinking（同一 agent 每秒最多推一次）、`usage_update` 的 used/size→`context_pct`、`session/request_permission`→`permission`（送出拒絕後清空）接到 Hub
  - **R-10 慢訂閱者默默漏轉換**：佇列溢出改送 `event: resync`；契約：前端收到就重抓 `GET /api/v1/activity`（`static/pixel/stream.js` 已實作）
- **tailer 閒置成本歸零**（問題單 T-1／T-2／T-5）：目錄 mtime 沒變就只 stat 活躍集合（最近 `ARK_KIRO_TAIL_MAX_AGE_DAYS`＝7 天有變動），每 60 秒定期全掃接住舊 session 續寫；3000 個舊檔閒置每輪 24ms → 0.0ms。全隊 claude-code 時 daemon 不啟動 tailer（log 說明原因）
- **rate_limit 改看結構化錯誤**（問題單 A-1）：ACP instance 的錯誤偵測只掃 JSON-RPC 錯誤回應與 stdout 非 JSON 行（stderr），不再掃 agent 自己的文字 —— 連「Error: rate limit reached」這種像錯誤的敘述也不會誤判；真正的 `API Error: 429 … rate_limit_error` 照樣判出。kiro PTY 沒有結構化來源，維持 2.1.7 收緊後的比對

## 2.1.8 (2026-11-06)

- **session tailer 輪詢成本**（問題單 `docs/issues/2026-11-06-session-tailer-full-scan-cpu.md` T-3／T-4）：3000 個未變動的舊檔，每輪 CPU 88ms → 30ms（profile：主因是每檔建 `Path` 再 `str()` 當快取鍵，不是 stat）—— 改用 `os.scandir` 回傳字串路徑＋一次 stat、不排序，只有變動的檔才建 `Path`；輪詢間隔改由 `ARK_KIRO_TAIL_POLL_S` 設定，預設 1 → **3 秒**（下限 0.2）。合計閒置 CPU 約降為原本 1/9。尚未做 T-1／T-2（目錄 mtime＋只追活躍檔，閒置趨近 0）與 T-5
- **啟動 log 不再誤報人格被換成範本**（kiro-bot 回報）：`Using SOUL.md template …`、`Generated agent JSON dynamically …` 原本在判斷 policy 前就印 info —— `once` 且檔案已存在時根本不寫，卻像 manager 人格被覆寫。改成：挑範本印 debug；真的寫檔才印 info 並帶來源（`Wrote SOUL.md: …（來源 …）`）；已存在印 debug「已存在（policy=once），保留」

## 2.1.7 (2026-11-06)

> Pixel Office 移植計畫 P3～P5（計畫 paddy-bot `docs/plans/2026-11-05-pixel-office-port-plan.md`；前端來自 nana `apps/team-website`）。**預設關閉**：`PIXEL_OFFICE=1` 才出現。

- **像素辦公室**（`PIXEL_OFFICE=1`）：後台首頁 `/app` 改成辦公室，每個 agent 依 role 入座（指揮中心／戰情室／研發艙／待命艙），即時顯示讀檔、寫檔、跑指令、委派、等待；點角色或名牌開右側對話抽屜（沿用 `/api/v1/chat/*`）。總覽移到 `/app/dashboard`，側邊欄多「辦公室」。旗標關閉時 `/app`、選單、路由與 2.1.6 相同
- 網站 `GET /api/v1/activity`、`/api/v1/activity/stream` 轉發 daemon 的 ActivityHub（2.1.4）；登入規則同其他監控 API；未登入（`CONSOLE_PUBLIC_READ=1`）帶 `public=1` 由 daemon 遮蔽工具細節；daemon 不在時回 `ok: false`／SSE `offline`，前端顯示離線並自動重連
- **team.yaml `avatar`**（instance 選填）：`palette`（0–5，衣服顏色）、`area`（佈局區域 id，優先於 role 分區）、`sprite`（目前只有 `builtin:chibi`）。不合法的值不擋啟動，丟掉該欄位並寫 warning 說可用值；activity 快照多 `avatar`
- **佈局覆寫**：部署端 `web/pixel/layouts/office.json`（`ark.pixel_layout.v1`）優先於內建佈局，經 `GET /pixel/layout.json` 提供；格式不合法改用內建並寫 warning
- **後台 Agents＋對話合一頁**（`PIXEL_OFFICE=1`，移植 nana 9a46351）：`/app/agents` 左欄成員名冊、右側對話；`/chat`、`/chat/<agent>`、`/app/chat*` 轉到 `/app/agents?a=&c=`；主選單拿掉「對話」。對話主體抽成 `_chat_app.html`（`/chat` 與後台共用，保留 2.1.5 的 artifact 卡與 `fmtMsg`）。旗標關閉時 `/chat` 獨立頁、`/app/agents` 表格照舊
- **辦公室抽屜檔案卡**改讀 2.1.5 的 artifact 訊息（`kind: file／report`；開啟走簽章連結、下載走 ID）。原本沿用 nana 的路徑解析 `/api/v1/chat/artifacts/resolve`（套件沒有）→ 卡片不出現、artifact 訊息顯示成原始 JSON
- favicon；端到端測試 `tests/website/e2e/`（需 playwright＋`OFFICE_URL`，否則 skip）
- **修 TG 標題行的 `<b>` 被跳脫**（2.1.2～2.1.6）：回覆第一行 ≤ 30 字會當標題加粗，這行直接跳脫、沒走 2.1.2 的還原 → 「⏰ <b>狀態更新</b>」寫在第一行照樣顯示字面標籤（11-06 paddy 維運管家）。2.1.2 的測試只驗 `_md_to_html`，沒驗實際送出的 `_format_reply_html`
- **rate_limit 誤判**（問題單 `docs/issues/2026-11-06-rate-limit-false-positive-on-agent-narration.md` A-2～A-4）：只認錯誤句型（`rate_limit_error`、`API Error: 429`、`429 … Too Many Requests`、`ThrottlingException`、行首 `Error:` 的 rate limit…），agent 敘述裡的 throttled／rate limiting 不再觸發冷卻與告警；告警建議依後端（claude-code：共用帳號額度、換 model 無效；其他：換輕量 model）
- **`deploy doctor` FR-036 不再誤報專案根 manager 的 `.mcp.json`**（同一問題單 A-5～A-7）：team server 的 `--instance`／`--role` 與該 instance 一致 → 不算衝突（新舊模組名都認）；身分不一致、沒有 team server 或讀不了照報。專案根（有 team.yaml）的訊息改成「讓身分一致或移除」，不再建議搬到 `agents/{name}/`。nana 實機設定驗證：0 筆
- 尚未包含：rate_limit 改用 ACP 結構化錯誤（問題單 A-1）、自訂 PNG 角色（FR-F4）、在辦公室核可授權（FR-E；daemon 的 ACP 目前收到 permission 一律直接回應，沒有待決佇列）、手機版版面

## 2.1.6 (2026-11-06)

> 原本標 2.1.5，但另一個 session 已先以 2.1.5 發布 Pixel Office P2（版號撞到）→ 改發 2.1.6，內容不變

- **使用者訊息的來源標頭**：daemon 轉給 agent 的使用者訊息，第一行一律是 `[來源：TG 私訊 · 名稱]`／`[來源：TG 群組 · #topic · 名稱]`／`[來源：web · 名稱]`（網站舊前綴 `[web · 名稱]` 由 daemon 正規化，nana 自有網站同樣適用）。原本 TG 只送原文，agent 分不出 TG 與 IDE（nana 小娜依來源判斷的規則永遠判成 IDE、只能靠補送）
- **修 `backend switch`／`status` 實機不能用**（2.1.1～2.1.4）：`/api/instances` 實際回 `{name: {...}}`，原本當 list 解析 → switch 寫檔前就 TypeError、status 全是 `?`；測試的假 API 改用實機形狀。讀不到狀態時改成顯示警告（原本默默吞掉）
- **TEAM.md 統一回覆規則**：「開頭有 `[來源：…]` → 一律呼叫 reply()；沒有 → IDE 直接對話，不要呼叫 reply()」—— 各部署不必再在 CLAUDE.md 自己寫

## 2.1.5 (2026-11-06)

> Pixel Office 移植計畫 P2（設計 paddy-bot `docs/designs/web-channel-artifacts-design.md`、問題單 `docs/issues/2026-11-05-web-reply-one-shot-followups-to-telegram.md`、`docs/issues/2026-11-05-reply-file-ignores-web-source.md`）。

- **web 一級通道**：網頁發問後，agent 同回合第二則以後的 `reply()`、派工鏈的延遲回報都回同一個網頁對話（原本第一則之後跑去 TG）。解綁條件：之後收到 TG／排程訊息、閒置超過 `communication.web_channel_ttl_s`（預設 1800；`0`＝2.1.4 一次性行為）、網頁關閉對話。turn 結束補發只在本回合沒 `reply()` 時發生
- `GET /api/web-reply/<conv>` 多回 `open`；`/api/health` 多 `web_sources`（instance → conv／age／回覆數）；日誌 `🌐 WEB REPLY #n`
- **網站續收**：`chat_relay` 第一批之後背景續收到 `open: false`；發問前不再 `DELETE` 佇列（會清掉未讀續報），改成先寫進對話。舊 daemon 沒有 `open` → 行為同 2.1.4
- **網頁傳檔（artifact）**：網頁對話中 `reply_file` 不送 TG，登記到 `state/artifacts.db`（白名單沿用 reply_file、resolve 後封閉檢查、sha256）並投 `kind: file` 訊息。新 API：`GET /api/artifacts/<id>`、`/download`、`/raw`、`POST /api/artifacts/<id>/link`（10 分鐘 HMAC 簽章）、`GET /r/<id>`（`Content-Security-Policy: sandbox allow-scripts allow-popups`，無 allow-same-origin）；過期 403、原檔刪除或被換 410。網站轉發 `/api/v1/chat/artifacts/*`、`/chat/r/<id>`（只限自己的對話），chat 顯示檔案卡（開啟／下載）
- **長答覆報告回網頁**：web 來源的長答覆達 `report.min_chars` → MD＋HTML 登記成 artifact，投 `kind: report` 報告卡（不需要 TG）

- **網頁對話顯示 TG HTML**：agent 回覆直接寫的 `<b>…</b>`（如維運日報）在網站 chat 泡泡顯示成字面標籤 → 改用 `fmtMsg()`：先全體跳脫，只還原同一行、成對、無屬性的 b／i／u／s／code（與 2.1.2 TG 端同規則）；對話串預覽去標籤

## 2.1.4 (2026-11-05)

> Pixel Office 移植計畫 P1（設計 paddy-bot `docs/designs/acp-tool-events-design.md`、問題單 `docs/issues/2026-11-05-acp-structured-tool-events-for-activity.md`）。

- **ACP 結構化工具事件**：`process_acp` 把 `tool_call`／`tool_call_update`／kiro `tool_call_chunk`（`session/update` 與 `_kiro.dev/session/update` 兩條通道）送成 `ToolEvent`（call_id、kind、title、status、遮罩過的 detail）；渲染行與 `_output_count`、補送、白名單行為不變
- 渲染行不再出現 `[acp] tool ?`：沒標題的 update 用同 call_id 第一次的標題
- **ActivityHub**（daemon）：每個 agent「正在做什麼」（讀檔／寫檔／跑指令／委派／思考），狀態判斷沿用 nana 先行版的純函式
- 新 API：`GET /api/v1/activity`（快照，`ark.agent_activity.v1`）、`GET /api/v1/activity/stream`（SSE，先快照後增量、15 秒 keepalive、`Last-Event-ID`）；`?public=1` 套 `redact_public`
- 網站／Pixel Office 不再需要每 2 秒輪詢 output（前端改用此 API 在 P3）

## 2.1.3 (2026-11-05)

- **角色分 4 層**（Paddy 定）：`/status`、啟動橫幅、`/help` 統一為 👑 指揮（manager）→ 🛠 維運（admin）→ 📋 專案（leader）→ ⚙️ 執行（worker）；沒成員的層不顯示。原本 admin 與 manager 併在「指揮」、`/help` 只分兩層。層名與圖示走訊息目錄（`cmd.status.label_*`／`icon_*`），可用 `messages.yaml` 改
- `.ark` 佈局：`.claude/skills` 是自備的技能目錄時不標 `adapter_degraded`（paddy 試點前發現）

## 2.1.2 (2026-11-05)

> **`.ark` 工具中立佈局 M1～M3**（計畫 paddy-bot `docs/plans/ark-workspace-layout-os-plan.md`，R-1～R-12 定案）：`.ark` 是真相、`.kiro`／`.claude` 是投影、`AGENTS.md` 是大門。**既有 `.kiro` 佈局行為不變**（backend 產出逐字快照比對）。本版先支援 Claude；kiro 轉接器在之後的 2.1.x。

### M1 路徑解析（行為不變）
- `ArkWorkspace`：判定 `ark`（有 `.ark/ark.yaml`）／`legacy_kiro`，各層路徑唯一入口；套件 77 處寫死的 `.kiro` 全部改經它，守門：再寫死就紅
- `team.yaml ark_files`（`kiro_files` 為別名）；`/api/health.workspace_layout`
- team MCP 啟動模組改 `ark_agent_os.team_mcp`（判定新舊都認）；`team_mcp_spec()` 純函式

### M2 Claude 轉接器
- `ark-agent-os ark sync [--check]`：根 CLAUDE.md 只動 `<!-- ark:begin/end -->` 受管區塊（手寫內容一字不動）、`.mcp.json`、`.claude/commands`、`.claude/skills` 連結；`.ark/adapters.lock` 記雜湊，`--check` 抓手改／缺檔
- agent 執行：`.ark` 佈局的 claude agent 從 `.ark/brain`＋`.ark/runtime/TEAM.md` 匯入、MCP＝`.ark/mcp.json`＋team MCP 即時注入（不讀 `.kiro`）
- IDE 身分 `<instance>@ide`：權限比照本人、不動本人的回覆狀態；reply 直送本人私訊
- 工具白名單：`.ark/agents/<name>.yaml` 的 `tools` → claude `session/new._meta.claudeCode.options.tools`；未宣告照舊（health `permission_unscoped`）
- daemon 啟動時自動投影；人寫同名檔不覆寫、標 `adapter_user_owned`

### M3 遷移
- `ark-agent-os ark migrate [--apply]`：`.kiro/steering`→`.ark/brain`（inclusion 抽進 `ark.yaml`）、TEAM.md 交 daemon 寫 `.ark/runtime/`、prompts、宣告的 MCP、技能複本；手寫 CLAUDE.md 只移除 `@.kiro/steering` 匯入行；**不碰 team.yaml**；先備份、失敗全部還原；有 kiro instance 的部署拒絕
- `backend switch`：`.ark` 佈局下不准切到含 kiro instance 的組

### 其他
- doctor：手寫但已匯入 SOUL／TEAM 的 CLAUDE.md 不算衝突（ninja 升 2.1.0 驗證誤報）
- agent 回覆裡直接寫的 TG HTML（`<b>…</b>` 等成對、同一行、無屬性的 b／i／u／s／code）正常顯示 —— 原本被跳脫成字面標籤（paddy 改 claude 後 11-05 TG 實測）

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
