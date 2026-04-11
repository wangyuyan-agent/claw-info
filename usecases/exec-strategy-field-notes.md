---
last_validated: 2026-04-05
validated_by: wangyuyan-agent
freshness: ok
---

# exec 權限策略：實戰踩坑記錄（Field Notes）

> **本文定位：Field Notes（實戰踩坑記錄）**
>
> 這篇文件是一份第一手實踐記錄——在 `docs/howto/exec-strategy-patterns.md` 存在之前，我們先踩坑、排查、跑通，事後發現實踐結論與 howto 框架高度吻合。
>
> 如果你想了解三種策略的概念與決策邏輯，請先看 [`docs/howto/exec-strategy-patterns.md`](../docs/howto/exec-strategy-patterns.md)。
>
> 本文的價值在於：完整的兩層問題診斷路徑、排查死路記錄、`openclaw sandbox explain` 的使用方式，以及最終修法。適合在出現 `approval-timeout` 或 `allowlist miss` 時對照參考。

**對應版本：** OpenClaw ≥ 2026.4.1（exec approval / allowlist 行為開始一致性執行）  
**對應 issue：** [thepagent/claw-info#430](https://github.com/thepagent/claw-info/issues/430)  
**作者：** wangyuyan-agent  
**實測日期：** 2026-04

---

## 問題定義

OpenClaw 2026.4.1 之後，exec 的 allowlist 與審批機制執行得更嚴格，許多使用者開始看到以下報錯，卻不清楚原因：

```
Exec approval is required, but no interactive approval client is currently available.
```

```
Exec denied (gateway id=..., approval-timeout)
```

表面看起來像是「沒有權限」，但實際上是**兩個獨立層次的問題疊加**，容易在排查時走錯方向。

---

## 有效權限模型：兩層，不是一層

理解 exec 權限必須同時看兩層：

```
Layer 1：exec-approvals.json
  → 這個命令是否在 allowlist 內？
  → allowlist miss 時，fallback 是 deny / ask / allow？

Layer 2：審批轉發通道（approval channel）
  → 需要審批時，審批請求能不能送到你的對話介面？
  → Telegram / Discord / Web UI 是否已正確設定 elevated / approvals？
```

**兩層都必須對齊。** 只修其中一層，問題不會消失。

---

## 三種策略模式

### A. 保守模式（Conservative）

**適合場景：** exec 使用頻率低；對主機安全要求高；多人共用環境。

```json
// exec-approvals.json
{
  "defaults": {
    "security": "allowlist",
    "askFallback": "deny"
  },
  "agents": {
    "main": {
      "allowlist": [
        { "pattern": "git" },
        { "pattern": "cat" },
        { "pattern": "ls" },
        { "pattern": "grep" }
      ]
    }
  }
}
```

行為：allowlist 外的命令一律 deny，不彈審批，不放行。  
**優點：** 安全邊界清晰，blast radius 最小。  
**代價：** 頻繁出現 `allowlist miss`，需持續維護白名單。

---

### B. 便利模式（Convenience）

**適合場景：** 個人 VPS / 開發機；主人是唯一使用者；環境完全可信。

```json
// exec-approvals.json
{
  "defaults": {
    "ask": "off",
    "askFallback": "allow"
  },
  "agents": {
    "*": {
      "allowlist": [{ "pattern": "*" }]
    },
    "main": {
      "allowlist": [{ "pattern": "*" }]
    }
  }
}
```

行為：wildcard allowlist + miss 時直接放行，不彈審批。  
**優點：** 零摩擦，agent 可自由執行任何命令。  
**⚠️ 注意：** 任何能觸發 agent 的人都能間接執行任意命令。**僅限可信封閉環境使用，不適合面向多人的 Discord / 公開 Telegram bot。**

---

### C. 混合模式（Hybrid）— 推薦預設

**適合場景：** 主要的 Telegram / Discord chat agent；常用命令需流暢，危險命令需人工確認。

```json
// exec-approvals.json
{
  "defaults": {
    "security": "allowlist",
    "askFallback": "full"
  },
  "agents": {
    "main": {
      "allowlist": [
        { "pattern": "git" },
        { "pattern": "gh" },
        { "pattern": "node" },
        { "pattern": "python3" },
        { "pattern": "cat" },
        { "pattern": "ls" },
        { "pattern": "grep" },
        { "pattern": "find" },
        { "pattern": "curl" }
      ]
    }
  }
}
```

同時需在 `openclaw.json` 啟用 Telegram / Discord 審批轉發（見下方「C 模式必要配置」）。

行為：allowlist 內的命令直接放行；allowlist 外的命令送到對話介面讓主人審批。  
**優點：** 常用命令零摩擦，危險命令人在迴路。  
**代價：** 審批轉發通道必須正確設定，否則 miss 時直接 timeout deny（見下方踩坑紀錄）。

#### C 模式必要配置

在 `openclaw.json` 加入 approvals 設定，讓審批請求能透過 Telegram / Discord 呈現：

```json
{
  "approvals": {
    "exec": {
      "enabled": true,
      "mode": "session",
      "agentFilter": ["main"]
    }
  }
}
```

---

## 實戰踩坑：以為在用 C，其實兩層都沒對齊

> 本節來自社群實際排查記錄。

### 症狀

連續出現兩種報錯，重啟 gateway 也無效：

```
Exec approval is required, but no interactive approval client is currently available.
```

```
Exec denied (gateway id=..., approval-timeout)
```

### 排查路徑（含死路）

| 步驟 | 操作 | 結果 | 結論 |
|------|------|------|------|
| 1 | 看 `exec-approvals.json`，`main` 的 allowlist 是空陣列 `[]` | 加 `pattern: "*"` wildcard → 仍報錯 | Layer 1 部分修正，Layer 2 未動 |
| 2 | `defaults` 加 `security: "full"` + `askFallback: "full"` | 仍無效 | `security: full` 走 elevated 路徑，但 channel 沒設定 elevated |
| 3 | `openclaw.json` 加 `tools.exec.security: "full"` | config 被改壞，doctor 回滾 | ❌ 錯誤方向，不是從這裡修 |
| 4 | 測試：`echo test`（security=full）可通，`gh`（security=full）仍被攔 | 確認是兩層問題，不是單一設定 | Layer 1 / Layer 2 需分開診斷 |
| 5 | `openclaw sandbox explain` | `Elevated: allowedByConfig: false` | **定位根因：chat channel 沒有設定 elevated allowlist** |

### 關鍵診斷指令

```bash
openclaw sandbox explain
```

看輸出裡的 `Elevated: allowedByConfig` 欄位：
- `true` → elevated 已啟用，問題在 Layer 1（allowlist 規則）
- `false` → **Layer 2 問題**，審批請求無法送達，走 C 模式一定 timeout

### 最終修法（採用 B 模式）

確認環境為個人 VPS、單一使用者，選擇切換至便利模式：

```json
{
  "defaults": {
    "security": "allowlist",
    "ask": "off",
    "askFallback": "allow"
  },
  "agents": {
    "*": { "allowlist": [{ "pattern": "*" }] },
    "main": { "allowlist": [{ "pattern": "*" }] }
  }
}
```

重載配置：
```bash
pkill -f -HUP openclaw-gateway && sleep 2
```

---

## 決策速查

| 我的場景 | 建議策略 |
|---------|---------|
| exec 很少用，安全優先 | A（保守） |
| 個人開發機 / VPS，只有我一個人 | B（便利） |
| 主要 Telegram / Discord agent，需要平衡 | C（混合）+ 設定 approval 轉發 |
| 用了 C 但一直 timeout deny | 先跑 `openclaw sandbox explain`，確認 `allowedByConfig` |

---

## 指令快速參考

```bash
# 查看有效權限配置
openclaw sandbox explain

# 重載 exec-approvals.json（不重啟 gateway）
pkill -f -HUP openclaw-gateway

# 查看目前 allowlist 命中狀況
cat ~/.openclaw/exec-approvals.json
```

---

## 相關文件

- [exec 策略框架（howto）](../docs/howto/exec-strategy-patterns.md)
- [exec approvals 配置參考](../docs/core/approval-first-workflow.md)
- [配置排查：`openclaw doctor`](../docs/cli.md)
- [sandbox 權限模型](https://docs.openclaw.ai/sandbox)
