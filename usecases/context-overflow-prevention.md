# Context Overflow 防護協議：避免 Agent Session 撐爆

> 一次真實事故的教訓：browser tool 如何把 session context 推到崩潰邊緣，以及怎麼不讓它發生。

---

## TL;DR

- **context 是 agent 的記憶上限**，超過就崩潰——`prompt too large` 或回應品質劇降
- **browser tool 是最大風險**：每次 snapshot 可能帶入數萬 token，且無法撤回
- **web_fetch + r.jina.ai** 是輕量替代：只取目標內容，單次 ≤ 3000 字
- **每次 browser/web_fetch 前先查 context 水位**：> 50% 立即停止
- **長任務每 5 步自動檢查一次**，不要等到崩才知道
- **出問題時用 `/reset`**，不要在已撐爆的 session 裡繼續掙扎

---

## 這個問題是怎麼發生的

2026-02-24，一個使用 `agent-browser` 的 session 任務途中 context 持續膨脹，反覆觸發 `prompt too large` 錯誤，需要手動介入才能救場。

根本原因：

1. browser snapshot 把完整 DOM 帶進 context
2. 每次操作都加一層，context 線性增長
3. 問題發現時已無法在同一 session 繼續

這不是小概率事件。任何需要讀多個頁面的任務都會遇到。

---

## 核心概念：Context 水位

```
Context 使用率
     0%  ────────────────────────────────────── 100%
          │          │           │              │
         正常        ⚠️ 警告     🛑 停止        💥 崩潰
         < 30%    30%-50%     > 50%
                              不再用
                              browser/
                              web_fetch
```

**關鍵原則**：context 不能回收。一旦寫進去的 token 就佔著，只有 `/reset` 能清空。

---

## 工具選擇：輕量 vs 重量

### 優先用 web_fetch + r.jina.ai

```bash
# ✅ 輕量：只取頁面的可讀文字，單次限 3000 字
web_fetch("https://r.jina.ai/https://example.com/page", maxChars=3000)
```

`r.jina.ai` 會把目標頁面轉成純文字 Markdown，剝掉 HTML/JS/CSS，大幅壓縮 token 使用。

### 什麼情況才用 browser

只有以下情況才需要 browser：

- 需要操作 UI（點擊、填表、登入）
- 頁面是 SPA，內容靠 JS 渲染，web_fetch 拿不到
- 需要截圖給人看

用 browser 時的限制：

```
每次 snapshot maxChars ≤ 3000
每次只提取目標欄位，不返回完整 DOM
```

---

## 操作規程

### 每次使用前：查水位

```
呼叫 session_status 確認 context 使用率

< 30%  → 繼續
30-50% → 警告使用者，建議 /reset 後繼續
> 50%  → 立即停止 browser/web_fetch 操作，告知使用者
```

### 長任務（> 5 次工具呼叫）：每 5 步自動檢查

```
Step 1 → Step 2 → Step 3 → Step 4 → Step 5
                                          │
                                          ▼
                                    查 context 水位
                                          │
                              ┌───────────┴───────────┐
                              │                       │
                           安全（< 30%）           警戒（≥ 30%）
                              │                       │
                           繼續                  告知使用者
```

### 分段提取長頁面

```bash
# ❌ 錯誤：一次拿整頁
web_fetch("https://r.jina.ai/very-long-page.com")

# ✅ 正確：分段，每段存到中間變數
web_fetch("https://r.jina.ai/very-long-page.com", maxChars=3000)
# 處理第一段，存下需要的資訊
# 繼續取下一段（若需要）
web_fetch("https://r.jina.ai/very-long-page.com?offset=3000", maxChars=3000)
```

---

## Session 急救

### 症狀識別

| 症狀 | 診斷 |
|------|------|
| `prompt too large` 錯誤 | context 已超限 |
| 回應開始重複、矛盾 | context 接近上限，模型開始「忘記」 |
| 工具呼叫無故失敗 | 可能是 context 過長導致指令被截斷 |
| 任務越做越慢 | token 計費增加，也是水位信號 |

### 救援步驟

```
1. 停止所有 browser/web_fetch 操作
2. 把當前進度摘要寫到檔案（memory/YYYY-MM-DD.md）
3. 通知使用者執行 /reset
4. Reset 後從摘要檔案恢復上下文繼續任務
```

### 最後手段（session 無法正常運作時）

```bash
# 清理僵尸 session 條目
python3 /root/clean_openclaw_session.py

# 歸檔超大 .jsonl 文件
bash /root/clean_openclaw_session.sh
```

---

## 最佳實務

**把需要記住的東西寫進檔案，不要靠 context 記**

```
# ❌ 依賴 context 記住中間結果
# （5步之後模型可能已經「忘了」）

# ✅ 重要中間結果寫到檔案
write("memory/task-progress.md", "目前進度：已完成步驟 1-3，下一步是...")
```

**任務開始前估算 context 消耗**

一個 web_fetch 大概消耗多少 token？粗略估算：
- 純文字 1000 字 ≈ 1500 token
- maxChars=3000 ≈ 4500 token
- browser snapshot（無限制）≈ 10000~50000 token

長任務需要讀 5 個頁面？先算好：5 × 4500 = 22500 token，再決定要不要 reset 後再開始。

---

## Anti-patterns

| 做法 | 問題 |
|------|------|
| 不限 maxChars 直接 web_fetch | 頁面可能幾萬字，全進 context |
| browser snapshot 後不 reset 繼續下一個任務 | context 帶著上個任務的 DOM 殘留 |
| 在已 > 50% 的 session 裡繼續長任務 | 遲早崩，而且崩的時候進度全丟 |
| 出錯後繼續在同一 session 裡重試 | 每次重試都加 token，加速崩潰 |

---

## 相關連結

- `docs/cron.md`：Cron isolated session 的 context 管理
- `usecases/cron-automated-workflows.md`：自動化任務設計
