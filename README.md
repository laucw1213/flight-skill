# ✈️ Flight Search Skill

透過 **Trip.com 深度 URL + agent-browser** 搜尋機票的 Claude Skill。

## 原理

```
構建 Trip.com 深度 URL → agent-browser 打開 → snapshot 提取航班數據 → 用戶選擇 → 協助訂票
```

無需 API Key，直接利用已登入的 Trip.com session 搜尋。

## 前置條件

### 1. 安裝 agent-browser
```bash
npm install -g @anthropic-ai/agent-browser
# 或
brew install agent-browser
```

### 2. 首次登入 Trip.com
```bash
agent-browser --session-name trip open "https://hk.trip.com/?locale=zh_hk&curr=HKD"
```
在彈出的瀏覽器視窗手動登入，完成後 cookies 自動儲存。

### 3. 確認登入成功
```bash
agent-browser --session-name trip snapshot | grep -E "會員|我的訂單"
```

## 使用方式

```
用戶：「幫我搵由香港去倫敦 3月9日 單程機票」
Claude：自動觸發 skill → 搜尋 Trip.com → 顯示結果 → 協助訂票
```

## 檔案結構

```
flight-skill/
├── SKILL.md    ← 主要 Skill 定義（Claude 執行指引）
└── README.md   ← 本檔案（安裝說明）
```

## Session 持久性

agent-browser 用 `--session-name trip` 保存 cookies，重啟後不需重新登入。
如 session 過期，重新執行首次登入步驟即可。
