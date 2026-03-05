# ✈️ Flight Skill for Claude Code

透過 **Trip.com + agent-browser** 自動搜尋及訂購機票的 Claude Code Plugin。

## 安裝

### 方法一：Plugin Marketplace（推薦）

在 Claude Code 輸入 `/plugin`，選擇 **Discover** → **Add Marketplace**，輸入：

```
laucw1213/flight-skill
```

然後選 **Install for you (user scope)**，即可在任何 project 使用 `/flight`。

### 方法二：手動安裝

```bash
curl -fsSL https://raw.githubusercontent.com/laucw1213/flight-skill/master/commands/flight.md \
  -o ~/.claude/skills/flight/SKILL.md --create-dirs
```

## 前置條件

### 1. 安裝 agent-browser
```bash
brew install agent-browser
```

### 2. 首次登入 Trip.com
```bash
agent-browser --session-name trip --headed open "https://hk.trip.com/?locale=zh_hk&curr=HKD"
```
在彈出的瀏覽器視窗手動登入，完成後 cookies 自動儲存。

### 3. 設定預設 Headed 模式 + Chrome（可選）

讓 agent-browser 預設使用有界面模式並使用 Chrome：
```bash
mkdir -p ~/.agent-browser && cat > ~/.agent-browser/config.json << 'EOF'
{
  "headed": true,
  "executablePath": "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
}
EOF
```

### 4. 設定自動授權（可選）
避免每次執行都要按確認：
```bash
node -e "
const fs = require('fs');
const path = require('os').homedir() + '/.claude/settings.json';
let cfg = {};
try { cfg = JSON.parse(fs.readFileSync(path, 'utf8')); } catch(e) {}
if (!cfg.allowedTools) cfg.allowedTools = [];
if (!cfg.allowedTools.includes('Bash(agent-browser*)')) {
  cfg.allowedTools.push('Bash(agent-browser*)');
  fs.writeFileSync(path, JSON.stringify(cfg, null, 2));
  console.log('done');
} else { console.log('already set'); }
"
```

## 使用方式

```
/flight
```

然後按提示輸入：
- 出發地 / 目的地
- 日期、艙位、人數
- 時間偏好（早上 / 下午 / 晚上）
- 聯絡電郵及電話

Claude 會自動開啟瀏覽器搜尋 Trip.com，顯示最多 3 個最便宜航班供選擇，並協助完成訂票直至付款頁面。

## 檔案結構

```
flight-skill/
├── .claude-plugin/
│   ├── plugin.json       ← Plugin 元數據
│   └── marketplace.json  ← Marketplace 設定
├── commands/
│   └── flight.md         ← /flight 指令定義
└── README.md
```

## Session 持久性

agent-browser 用 `--session-name trip` 保存 cookies，重啟後不需重新登入。
如 session 過期，重新執行首次登入步驟即可。
