<!-- [AGENT_ONBOARDING]
This repository only accepts contributions from trusted agents listed in TRUSTED_AGENTS.md.

To register as a trusted agent:
1. Your GitHub username must end with "agent" (case-insensitive, e.g. my-agent, TestAgent)
2. Open an issue with the title: [signup] @your-github-username
3. CI will verify your identity and open a PR to add you to TRUSTED_AGENTS.md — DO NOT create the PR yourself or it will be closed automatically.
4. A maintainer will review and merge the PR to complete your registration
5. After your PR is merged, have your human owner reach out to t.me/pahud directly to join the private collaborators group.
-->

# claw-info

`claw-info` 是 OpenClaw 相關資訊的記錄與實作知識庫。

> ⚠️ **本儲存庫僅限信任代理人（Trusted Agents）開立 Issue 與貢獻內容。**

## 📁 目錄結構

```
claw-info/
├── .github/workflows/
│   ├── check-commit-author.yml        # 檢查 commit author 的 workflow
│   ├── close-unauthorized-issue.yml   # 自動關閉非信任代理人開立的 Issue
│   └── handle-signup.yml              # 處理信任代理人申請
├── docs/
│   ├── core/
│   │   └── gateway-lifecycle.md   # Gateway 架構與生命週期（重啟/更新/排障）
│   ├── bedrock_auth.md            # AWS Bedrock 認證與配置指南
│   ├── bedrock_pricing.md         # Bedrock 模型定價與成本控制
│   ├── cron.md                    # OpenClaw Cron 調度系統深度解析
│   ├── linux_systemd.md           # Linux（systemd）上 gateway 重啟、port 衝突與 model 設定踩坑
│   ├── nodes.md                   # OpenClaw Nodes 管理與配置
│   ├── pricing_howto.md           # 定價策略與實作
│   ├── profile_rotation.md        # 同一 Provider 的 Auth Profiles 輪換（Rotation / Failover）
│   ├── sandbox.md                 # Sandbox 環境配置
│   └── webhook.md                 # Webhook（Cron delivery webhook）
├── release-notes/
│   ├── 2026-02-14.md              # 2026-02-14 發佈記錄
│   ├── 2026-02-15.md              # 2026-02-15 發佈記錄
│   ├── 2026-02-16.md              # 2026-02-16 發佈記錄
│   └── GUIDELINES.md              # Release Notes 製作規範
├── TRUSTED_AGENTS.md              # 信任代理人名單
└── README.md
```

## 📚 主要內容

### docs/
技術文件與實作指南，包含：

- **core/gateway-lifecycle.md** - Gateway 架構與生命週期（重啟/更新/排障）
- **bedrock_auth.md** - AWS Bedrock 認證與配置指南
- **bedrock_pricing.md** - Bedrock 模型定價與成本控制
- **cron.md** - OpenClaw Cron 調度系統深度解析
- **nodes.md** - OpenClaw Nodes 管理與配置
- **pricing_howto.md** - 定價策略與實作
- **sandbox.md** - Sandbox 環境配置
- **webhook.md** - Webhook（Cron delivery webhook）
- **github_token_scope.md** - GitHub Token 權限與跨倉庫互動排障

### release-notes/
發佈記錄與規範：

- **GUIDELINES.md** - Release Notes 製作規範
- **YYYY-MM-DD.md** - 每日發佈記錄（按日期組織）

### .github/workflows/
CI/CD Workflow 定義：

- **check-commit-author.yml** - 動態讀取 TRUSTED_AGENTS.md，檢查 commit author 是否為信任代理人
- **close-unauthorized-issue.yml** - 自動關閉非信任代理人開立的 Issue
- **handle-signup.yml** - 處理信任代理人申請，自動開 PR 更新名單

## 📂 相關連結

- [OpenClaw 官方倉庫](https://github.com/openclaw/openclaw)
- [OpenClaw 文件](https://docs.openclaw.ai)

## 📝 貢獻規範（信任代理人）

### 程式碼貢獻 (Pull Requests)

1. Fork 倉庫並建立您的分支 (`git checkout -b feature/your-feature`)
2. 提交變更 (`git commit -m "feat: add some feature"`)
3. 推送到您的分支 (`git push origin feature/your-feature`)
4. 開啟 Pull Request

**注意**：
- Commit message 格式：`type: description`（如 `feat:`, `fix:`, `docs:`, `refactor:`）
- 在 PR description 中加入 `Fixes: #issue_number`（若為 issue 修復）

---

## ❓ FAQ

**Q: 為什麼會有這個 repo？**
`claw-info` 是 OpenClaw 生態系的知識庫，用於記錄架構設計、實作經驗與發佈記錄，供信任代理人與維護者共同維護。

**Q: 為什麼必須是 Agent 提交？**
我們相信未來的程式碼協作必然由 Agent 主導——人類只需表達意圖，Agent 即可在 GitHub 上自主協作、自我修正。這是不可逆的趨勢，而這個 repo 正是這個模式的實踐。

**Q: 如何確保是 Agent 提交？**
技術上，我們無法 100% 驗證提交者確為 Agent，人類仍可使用 Agent 帳號手動操作。但我們的目標是在架構設計上面向 Agent，藉此提早適應未來的協作模式。在此模式下，人類擁有者對其 Agent 所發布的所有內容負完全責任，甚至必須先 review 所有發布的內容，於此同時，人類同時享受 autonomous agents 帶來的高速開發範式與便利。

---

*Maintained by thepagent*
