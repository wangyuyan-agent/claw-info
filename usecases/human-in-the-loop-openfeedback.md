# Human-in-the-Loop 審批閘門：openfeedback 實戰指南

> 讓自動化腳本在執行高風險操作前，先用 Telegram 向人類請示。

---

## TL;DR

- **問題**：自動化腳本跑起來就停不住，沒有人工審批的插槽
- **解法**：`openfeedback` 工具——自動化腳本暫停、發 Telegram 訊息給你、等你按核准或拒絕、根據回應決定繼續或中止
- **適合**：Coding agent 提交 PR 前確認、部署腳本執行前、任何「做了就難回頭」的操作
- **核心原理**：靠 exit code——approved → `exit 0`，rejected → `exit 1`，腳本用 `&&` 或 `if` 接住
- **安裝**：需從源碼編譯（Rust），設定一次之後透明使用

---

## 解決的問題

自動化流水線（cron 任務、coding agent、CI/CD）有時會做出難以撤銷的操作：

- 提交並 push commit
- 開 PR、發 issue 留言
- 部署到生產環境
- 刪除資料或覆蓋設定

沒有人工閘門，出錯就是出錯。`openfeedback` 在這些操作前插入一個 Telegram 確認步驟，讓人類保持最終決定權。

---

## 核心概念

### 工作流程

```
自動化腳本
     │
     │ 執行到高風險操作前
     ▼
┌─────────────────────────────┐
│  openfeedback "操作描述"     │
│  → 發 Telegram 訊息給你      │
│  → 等待你的回應              │
└────────────┬────────────────┘
             │
    ┌────────┴────────┐
    │                 │
    ▼                 ▼
 核准（✅）         拒絕（❌）
 exit 0            exit 1
    │                 │
    ▼                 ▼
繼續執行          中止，可附帶
                  拒絕原因文字
```

### Exit Code 語義

| 結果 | Exit Code | 腳本行為（用 `&&`） |
|------|-----------|-------------------|
| 核准 | 0 | 繼續執行後續指令 |
| 拒絕 | 1 | 後續指令不執行 |
| 超時 | 1 | 視為拒絕，安全預設 |

---

## 安裝步驟

### 前置條件

- Rust 工具鏈（`rustup`）
- 一個 Telegram Bot Token（`BotFather` 申請）
- 你的 Telegram chat_id

### 1. 編譯安裝

```bash
# 從 GitHub 取得源碼
git clone https://github.com/antx-code/openfeedback /tmp/openfeedback
cd /tmp/openfeedback

# 編譯並安裝到 ~/.cargo/bin/
cargo install --path .

# 確認安裝成功
openfeedback --version
```

未來更新：

```bash
cd /tmp/openfeedback && git pull && cargo install --path .
```

### 2. 建立設定檔

```bash
mkdir -p ~/.config/openfeedback
cat > ~/.config/openfeedback/config.toml << 'EOF'
bot_token = "<YOUR_BOT_TOKEN>"
chat_id = <YOUR_CHAT_ID>
trusted_user_ids = [<YOUR_CHAT_ID>]
locale = "zh-TW"
default_timeout = 60  # 秒，超時視為拒絕

[audit_log]
path = "~/.local/share/openfeedback/audit.jsonl"
EOF
```

> ⚠️ `bot_token` 與 `chat_id` 是敏感資訊，不要 commit 進 repo。

### 3. 驗證設定

```bash
# 測試審批流程（你應該會收到 Telegram 訊息）
openfeedback "測試：這是一個審批請求"
echo "exit code: $?"
```

核准後應看到 `exit code: 0`，拒絕後 `exit code: 1`。

---

## 使用方式

### 基本用法

```bash
# 在高風險指令前插入審批
openfeedback "即將執行：git push origin main（共 3 個 commit）" && \
  git push origin main
```

### 在 Shell 腳本中使用

```bash
#!/bin/bash
set -e

# 準備工作（無需審批）
git add .
git commit -m "feat: 新功能實作"

# 審批閘門
if openfeedback "準備 push 到 main，確認要繼續？"; then
  git push origin main
  echo "Push 完成"
else
  echo "已拒絕，push 取消"
  exit 1
fi
```

### 在 Coding Agent 流程中使用

讓 agent 在提交 PR 前請示：

```bash
# agent 的執行腳本片段
openfeedback "Agent 準備開 PR：${PR_TITLE}
分支：${BRANCH}
變更檔案數：${CHANGED_FILES}
是否核准？" && \
  gh pr create --title "${PR_TITLE}" --body "${PR_BODY}"
```

### 自訂 Timeout

```bash
# 重要操作給更長的等待時間（秒）
openfeedback --timeout 300 "生產環境部署確認" && deploy.sh
```

---

## 最佳實務

**審批訊息要寫清楚**：「繼續？」沒有意義。寫清楚「做什麼、影響什麼、有沒有退路」。

```bash
# ❌ 沒用的審批訊息
openfeedback "繼續執行？"

# ✅ 有用的審批訊息
openfeedback "準備刪除 S3 bucket: my-prod-backups-2024
此操作不可逆。
確認要繼續？"
```

**超時預設為拒絕**：這是安全的預設值。如果你不在、沒看到訊息，操作不會自動繼續。

**保留 audit log**：`config.toml` 裡設定 `audit_log.path`，所有審批記錄（包含拒絕原因）都會寫入，方便事後追蹤。

**不要在循環裡審批**：如果一個操作要審批 100 次，設計就錯了。批次操作應該一次審批整個計劃，而不是逐項確認。

---

## Anti-patterns

| 做法 | 問題 | 替代方案 |
|------|------|---------|
| 在 cron 排程的無人值守任務裡插 openfeedback | timeout 後一律拒絕，任務永遠無法完成 | 分開：需人工確認的用 cron 觸發後等待，純自動化的不插閘門 |
| 審批訊息不帶上下文 | 收到訊息不知道在問什麼 | 把操作描述、影響範圍都寫進訊息 |
| hardcode timeout 為極短值 | 手邊沒手機就超時 | config 裡設合理預設，高風險操作用 `--timeout` 延長 |

---

## Troubleshooting

**症狀**：`openfeedback` 發出訊息但按鈕沒有反應
- **可能原因**：Bot Token 設定錯誤，或 bot 未加入對話
- **處理方式**：先對 bot 發 `/start`，確認 bot 回應後再試

**症狀**：`exit code: 1` 但沒有收到 Telegram 訊息
- **可能原因**：`chat_id` 設定錯誤，或網路問題
- **處理方式**：`openfeedback --debug "test"` 查看連線狀態

**症狀**：想在 CI 環境中使用，但沒有辦法互動
- **可能原因**：CI runner 無法接收 webhook 回應
- **處理方式**：openfeedback 需要有人在 Telegram 端回應，純 CI 環境不適用；改用環境變數控制是否啟用閘門

---

## 安全注意事項

- `trusted_user_ids` 只填你自己的 chat_id，防止其他人審批你的操作
- Bot Token 不要出現在任何 log 或版本控制中
- audit log 檔案包含操作內容，注意存放路徑的權限

---

## 版本與依賴

- 從源碼編譯，無版本號管理，以 git commit 追蹤
- 依賴：Rust stable、Telegram Bot API

---

## 相關連結

- `usecases/cron-automated-workflows.md`：OpenClaw cron 與自動化排程
- `docs/cron.md`：Cron 系統深度解析
