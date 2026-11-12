# ark-agent-os

[![version](https://img.shields.io/badge/version-2.1.20-orange)](https://github.com/igs-paddyyang-tw/ark-agent-os/releases/tag/v2.0.0)

統一的 AI agent daemon：一個 daemon，以 `team.yaml` 的 profile 區分 **bot**（單一 manager）與 **team**（manager＋admin＋leader＋workers）。
**ark-team-agent 1.13.23 的後繼**（同一份程式碼）；`ark_team_agent`／`ark_bot_agent` 以相容 shim 保留到 M5。

## 安裝

```bash
pip install https://github.com/igs-paddyyang-tw/ark-agent-os/releases/download/v2.1.20/ark_agent_os-2.1.20-py3-none-any.whl
```

## 從 ark-team-agent 1.13.23 升級

```bash
ark-team-agent deploy upgrade --to 2.0.0
```

詳見 [CHANGELOG](CHANGELOG.md)。
