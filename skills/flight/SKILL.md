---
name: flight-search
description: >
  AI 機票搜尋助手 — 透過 Trip.com 深度 URL + agent-browser 搜尋最平機票。
  當用戶提及以下任何內容時自動觸發：機票、航班、飛機、flights、
  訂機票、最平機票、去旅行、fly to、出發地/目的地/日期、
  HKG/NRT/SIN 等機場代碼、Cathay Pacific、Trip.com、國泰。
  即使用戶只說「我想去東京」或「幫我搵機票」也應觸發此 Skill。
compatibility:
  tools:
    - agent-browser  # required: persistent browser session for Trip.com
---

# ✈️ Flight Search Skill

透過 **Trip.com 深度 URL + agent-browser** 搜尋機票，提取航班資料，讓用戶選擇並完成訂票。

## 工作流程

```
Setup:  首次使用設定  → 一次性設定 agent-browser 權限（之後不再問）
Step 1: 收集旅遊資料   → 問清楚出發地、目的地、日期、乘客數
Step 2: 搜尋 Trip.com  → 用深度 URL 打開 Trip.com，等待結果載入
Step 3: 提取航班資料   → 用 snapshot 讀取所有航班選項
Step 4: 顯示結果       → 列出最佳選項，讓用戶選擇
Step 5: 引導訂票       → 幫用戶在 Trip.com 完成訂票流程
```

---

## Setup：首次使用設定（只需一次）

檢查用戶是否已設定 `agent-browser` 權限：
```bash
grep -q "agent-browser" ~/.claude/settings.json 2>/dev/null && echo "already set" || echo "not set"
```

如果 **not set**，詢問用戶：

```
為了讓機票搜尋流程順暢，需要預先授權 agent-browser 指令（否則每個步驟都需手動點擊確認）。

請選擇授權方式：
① 自動設定（推薦）— 幫你加入 ~/.claude/settings.json，全局生效，之後所有項目都無需再確認
② 手動點擊 — 每次指令彈出時自己點「Always allow」，不修改任何設定
```

如用戶選 **①**，執行：
```bash
# 讀取現有設定（如有）並加入 agent-browser 授權
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

告知用戶：✅ 已設定完成，之後使用此 Skill 無需再點擊確認。

---

## Step 1：收集用戶資料（搜尋前一次問齊）

目標：**在搜尋前收集所有必要資料**，之後 Claude 全自動完成訂票，用戶只需在付款頁面輸入信用卡。

已提供的欄位不需再問。將未提供的欄位一次過列出詢問。

### 必填欄位
| 欄位 | 例子 | 備注 |
|------|------|------|
| 出發地 | HKG | IATA 機場代碼 |
| 目的地 | NRT | IATA 機場代碼 |
| 出發日期 | 2026-04-15 | YYYY-MM-DD |
| 行程類型 | 單程 / 來回 | oneway / return |
| 聯絡電郵 | xxx@gmail.com | 接收訂單確認 |
| 聯絡手機 | 56247318 | 香港號碼，預設 +852 |

### 選填欄位（有預設值）
| 欄位 | 預設 | 可選值 |
|------|------|--------|
| 回程日期 | — | 來回必填 |
| 成人數 | 1 | 1-9 |
| 兒童數 | 0 | 0-8 |
| 艙位 | 經濟艙 (y) | y=經濟, s=豪華經濟, c=商務, f=頭等 |
| 直飛偏好 | 均可 | 直飛優先 / 均可 |
| 貨幣 | HKD | HKD / USD 等 |

### 行李（必填）
| 欄位 | 例子 | 備注 |
|------|------|------|
| 託運行李件數 | 0 | 0 = 不需要，1 = 一件 |

### 時間偏好（必填）
| 欄位 | 選項 | 備注 |
|------|------|------|
| 出發時間段 | 凌晨 / 早上 / 下午 / 晚上 / 不限 | 凌晨=00-07, 早上=07-13, 下午=13-19, 晚上=19-24 |
| 回程時間段 | 凌晨 / 早上 / 下午 / 晚上 / 不限 | 來回程才問 |

⚠️ 時間偏好為必填，避免搜尋結果過多。

### 常用機場代碼
```
香港: HKG    東京成田: NRT    東京羽田: HND
新加坡: SIN  倫敦希斯路: LHR  首爾仁川: ICN
台北: TPE    曼谷: BKK        吉隆坡: KUL
上海浦東: PVG  雪梨: SYD     洛杉磯: LAX
大阪關西: KIX  福岡: FUK      名古屋: NGO
```

### 城市英文名對照（用於 Trip.com URL）
```
HKG → hongkong     LHR → london       NRT → tokyo
SIN → singapore    ICN → seoul        TPE → taipei
BKK → bangkok      KUL → kualalumpur  PVG → shanghai
SYD → sydney       LAX → losangeles   KIX → osaka
```

---

## 航空公司選擇規則（Step 1 收集資料後確定）

### 規則一：來回程 → 必選國泰（CX）

來回程不作比較，**直接只搜尋 / 推薦國泰**。

### 規則二：單程 / 多程 → 預設國泰，除非便宜 ≥30%

1. 搜尋 Trip.com 結果，記錄：
   - **CX 最低價格**（若有直航，優先取直航 CX 價）
   - **所有非 CX 最低價格**（含廉航）
2. 計算：`非CX最低 / CX最低 × 100`
   - 若 **< 70%**（即便宜超過 30%）→ 推薦非 CX，在 Trip.com 訂票
   - 若 **≥ 70%** → 推薦 CX，在 Trip.com 訂票

```
例子：CX = HK$2,000；HK Express = HK$1,300（65%）→ 選 HK Express ✅
例子：CX = HK$2,000；Cathay Affiliate = HK$1,500（75%）→ 選 CX ✅
```

### 廉航（LCC）識別

以下航空視為廉航：
| 代碼 | 航空公司 |
|------|---------|
| UO | HK Express |
| AK / FD / QZ | AirAsia |
| TR | Scoot |
| JQ / 3K | Jetstar |
| MM | Peach |
| 5J | Cebu Pacific |
| VJ | VietJet Air |
| SL | Thai Lion Air |

其他非全服務航空亦視為廉航。國泰（CX）、國泰城（KA）、港龍、全服務亞洲航空（TG、SQ、MH 等）均屬全服務。

---

## Step 2：搜尋 Trip.com

### Trip.com 深度 URL 格式

**單程：**
```
https://hk.trip.com/flights/{from_city}-to-{to_city}/tickets-{FROM}-{TO}/?dcity={from_lower}&acity={to_lower}&ddate={date}&adult={adults}&child={children}&infant=0&cabin={cabin}&curr=HKD&lang=zh-hk
```

**來回：**
```
https://hk.trip.com/flights/{from_city}-to-{to_city}/tickets-{FROM}-{TO}/?dcity={from_lower}&acity={to_lower}&ddate={outbound_date}&rdate={return_date}&adult={adults}&child={children}&infant=0&cabin={cabin}&curr=HKD&lang=zh-hk
```

### 實際例子
```
# HKG → LHR 單程 2026-03-09 成人1名 經濟艙
https://hk.trip.com/flights/hongkong-to-london/tickets-HKG-LHR/?dcity=hkg&acity=lhr&ddate=2026-03-09&adult=1&child=0&infant=0&cabin=y&curr=HKD&lang=zh-hk

# HKG → NRT 來回 2026-04-15~22 成人2名 商務艙
https://hk.trip.com/flights/hongkong-to-tokyo/tickets-HKG-NRT/?dcity=hkg&acity=nrt&ddate=2026-04-15&rdate=2026-04-22&adult=2&child=0&infant=0&cabin=c&curr=HKD&lang=zh-hk
```

### 艙位代碼
| 艙位 | 代碼 |
|------|------|
| 經濟艙 | y |
| 豪華經濟艙 | s |
| 商務艙 | c |
| 頭等艙 | f |

### 執行搜尋
```bash
agent-browser --session-name trip open "{TRIP_COM_URL}"
```

等待 6-8 秒讓頁面載入完成，然後截圖確認結果已載入。

### 行李篩選器（載入後執行）

Trip.com 搜尋結果頁有「已包括託運行李限額」篩選器，勾選後**顯示的機票價已包含1件行李費**。

**用戶需要 0 件行李：**
不需操作篩選器，直接讀取結果（基本票價）。

**用戶需要 1 件行李：**
```bash
# 用 snapshot 找「已包括託運行李限額」篩選器並點擊
agent-browser --session-name trip snapshot | grep -i "行李"
agent-browser --session-name trip click {已包括託運行李限額 filter ref}

# 等待結果更新
agent-browser --session-name trip screenshot /tmp/trip_results_baggage.png
```
勾選後，結果頁的價格已包含1件行李，Step 4 顯示的即為含行李票價。

---

## Step 3：提取航班資料

### 截圖確認
```bash
agent-browser --session-name trip screenshot /tmp/trip_results.png
```

確認截圖顯示航班列表（而非載入畫面）。

### 用 Snapshot 提取數據
```bash
agent-browser --session-name trip snapshot
```

從 snapshot 提取以下資料（格式參考）：
```
'group "航班於{日期} {出發時間}由{出發機場}起飛，並於{日期} {抵達時間}抵達{目的地機場}。 飛行時間為{時長}，包括在{中轉城市}中途停留{候機時間} 單程價格為HK${價格}"'
```

### 提取欄位
| 欄位 | 位置 |
|------|------|
| 航空公司 | `region "{airline}"` |
| 出發時間 | group 描述文字內 |
| 抵達時間 | group 描述文字內 |
| 總飛行時長 | `group "{X}小時{Y}分鐘"` |
| 中轉城市 | tooltip 文字 |
| 候機時長 | tooltip 文字 |
| 價格 | `textbox "單程/來回價格為HK${price}"` |
| 剩餘座位 | 按鈕旁文字 `剩<9張` |

### 按時間偏好篩選（用 Trip.com 出發時間 Slider）

**不在 snapshot 手動過濾**，直接用 Trip.com 的出發時間 slider 限制範圍，結果頁只顯示指定時段航班。

| 時間段 | Slider 範圍 |
|--------|------------|
| 凌晨 | 00:00 – 07:00 |
| 早上 | 07:00 – 13:00 |
| 下午 | 13:00 – 19:00 |
| 晚上 | 19:00 – 24:00 |
| 不限 | 不操作 slider |

```bash
# 1. 取得 slider 軌道尺寸（用 $'...' 避免引號衝突）
agent-browser --session-name trip eval $'(function(){ var s=document.querySelectorAll("[role=slider]"); var t=s[0].parentElement.getBoundingClientRect(); return t.left+","+t.width+","+(t.top+t.height/2); })()'
# 例子輸出：57.5,267.27,548.36

# 2. 計算並拖曳 start/end slider（代入實際 x, w, y 數值）
agent-browser --session-name trip eval $'(function(){ var s=document.querySelectorAll("[role=slider]"); var x={X},w={W},y={Y}; var startH={START_HOUR},endH={END_HOUR}; function drag(el,h){ var tx=x+(h/24)*w; el.dispatchEvent(new MouseEvent("mousedown",{bubbles:true,clientX:el.getBoundingClientRect().left,clientY:y})); document.dispatchEvent(new MouseEvent("mousemove",{bubbles:true,clientX:tx,clientY:y})); document.dispatchEvent(new MouseEvent("mouseup",{bubbles:true,clientX:tx,clientY:y})); } drag(s[0],startH); drag(s[1],endH); return "set "+startH+":00-"+endH+":00"; })()'
```

**⚠️ slider 軌道尺寸因頁面不同而異，每次必須先讀取再計算，不可寫死。**

**如果篩選後結果為 0：**
每次放寬 2 小時（start -2, end +2），重複直到出現 ≥1 個結果為止。
找到結果後，告知用戶：
```
⚠️ 指定時間段（下午 13:00-19:00）沒有航班，已自動放寬至 11:00-21:00，找到以下航班：
```
有 ≥1 個結果即停止放寬，正常顯示航班列表。

---

## Step 4：顯示結果

### 航班推薦邏輯（顯示前先判斷）

**來回程：** 只顯示 CX 選項，其他航空不列出。

**單程 / 多程：**
1. 找出 CX 最低價 + 非 CX 最低價
2. 計算是否便宜 ≥30%
3. 在結果頂部標示推薦決定並說明原因

### 顯示數量規則

篩選後按價錢由低至高排列，**最多顯示 3 個航班**：
- ≥3 個結果 → 只顯示最便宜的 3 個
- 2 個結果 → 顯示 2 個
- 1 個結果 → 顯示 1 個

### 標準輸出格式（單程例子）

顯示票價（0件行李 = 基本票價；1件行李 = 含行李票價，已透過 Trip.com 篩選器處理）。

```
✈️ HKG → TPE | 2026-03-09 | 1位成人 | 經濟艙 | 行李：1件

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🏆 AI 推薦：HK Express UO112
原因：比國泰便宜 35%（超過30%門檻）

① 國泰航空 CX251
   23:00 HKG → 05:25+1 LHR | 直航 14h25m
   💰 HK$28,815

② HK Express UO112 ⭐ 推薦
   12:15 HKG → 13:55 TPE | 直航 1h40m
   💰 HK$956（比國泰便宜 35%）

③ ...
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

請問想選哪個航班？（輸入序號，直接 Enter 選推薦）
```

**來回程格式：**
```
✈️ HKG → NRT 來回 | 必選國泰（來回程規則）

① 國泰航空 CX543
   ...
```

---

## Step 5：引導訂票

### 5a. 點選航班

從 snapshot 找對應航班的「選擇」按鈕 ref：
```bash
agent-browser --session-name trip snapshot | grep -A2 "HK\${price}" | grep "button.*選擇"
agent-browser --session-name trip click {ref_id}
```

**來回程 — 回程航班篩選：**
選完去程後，Trip.com 顯示回程頁面。
⚠️ **絕對不要按「套用」按鈕** — 它會把去程時間篩選器複製到回程，導致 0 結果！
應手動操作：
1. 點擊「國泰航空」checkbox
2. 用 slider 設定回程時間偏好（與去程獨立設定）

### 5b. 艙位選擇彈窗

Trip.com 會彈出艙位選項（基本經濟 vs TripFlex 安心退改）。選最便宜選項，然後選「任何付款方式」，再點「繼續」。

⚠️ 繼續按鈕可能被 dialog 擋住，需用 JS 點擊：

```bash
# 選「任何付款方式」
agent-browser --session-name trip eval $'(function(){ var el = Array.from(document.querySelectorAll("*")).find(function(e){ return e.children.length === 0 && e.textContent.trim() === "任何付款方式"; }); if(el){ var p = el.closest("[class*=pay]") || el.parentElement; p.click(); return "clicked"; } return "not found"; })()'

# 點繼續（用 JS，避免被 dialog 擋住）
agent-browser --session-name trip eval $'(function(){ var btns = Array.from(document.querySelectorAll("button")); var cont = btns.filter(function(b){ return b.textContent.trim() === "繼續"; }); if(cont.length){ cont[0].click(); return "clicked"; } return "not found"; })()'
```

### 5d. 旅客資料頁（Step 1/4）

頁面分 4 步：**填寫資料 → 選座位 → 訂製行程 → 完成付款**

Trip.com 已儲存登入帳戶的乘客資料，以卡片形式列出。
用 JS 點擊選擇乘客（React 自定義 checkbox，需用 JS click）：
```bash
agent-browser --session-name trip eval '(() => {
  const els = document.querySelectorAll("*");
  for (const el of els) {
    if (el.textContent.trim() === "{PASSENGER_NAME}" && el.children.length === 0) {
      el.closest("[class*=pax-item]")?.click();
      return "clicked";
    }
  }
})()'
```

### 5e. 聯絡人資料

使用 Step 1 已收集的電郵和手機，直接填入（無需再問用戶）：

⚠️ 使用 `hk.trip.com` 時，國家代碼已預設 +852，無需更改。

```bash
# 填電郵
agent-browser --session-name trip fill {email_ref} "{user_email}"

# 填手機號（+852 已預填）
agent-browser --session-name trip fill {phone_ref} "{user_phone}"
```

如國家代碼非 +852，需手動更改：
```bash
agent-browser --session-name trip click {country_code_dropdown_ref}
agent-browser --session-name trip fill {country_search_ref} "Hong Kong"
agent-browser --session-name trip eval 'document.querySelectorAll("li").forEach(function(el){ if(el.textContent.indexOf("+852") !== -1){ el.click(); } })'
```

點下一步（可能需用 JS）：
```bash
agent-browser --session-name trip eval $'(function(){ var btns = Array.from(document.querySelectorAll("button")); var next = btns.find(function(b){ return b.textContent.trim() === "下一步"; }); if(next){ next.click(); return "clicked"; } return "not found"; })()'
```

### 5e2. 彈窗處理（可能出現）

**行李保障彈窗**（點 Next 前可能被阻擋）：
```bash
agent-browser --session-name trip snapshot | grep -i "No thanks\|不用了"
agent-browser --session-name trip click {No thanks ref}
```

**行李加購彈窗**（已用 Trip.com 篩選器處理，一律略過）：
```bash
agent-browser --session-name trip snapshot | grep -i "No thanks\|不用了"
agent-browser --session-name trip click {No thanks ref}
```

### 5f. 座位選擇（Step 2/4）

**所有航空統一規則：永遠選走廊位、最前排、沒有價錢上限。**

座位地圖在 **iframe** 內，需用 JS 訪問。

#### 第一步：自動偵測座位佈局（只看可用座位）

⚠️ 座位圖會同時顯示所有艙位，必須只看用戶所選艙位的可用座位（`cursor:pointer`）。

```bash
agent-browser --session-name trip eval '(function() { var doc = document.querySelector("iframe").contentDocument; var available = Array.from(doc.querySelectorAll("div.seat-item")).filter(function(el) { return el.style.cursor === "pointer"; }); var byRow = {}; available.forEach(function(el) { var t = el.textContent.trim(); var m = t.match(/^(\d+)([A-F-J-K]+)$/); if(m) { var r = parseInt(m[1]); if(!byRow[r]) byRow[r]=[]; byRow[r].push(m[2]); } }); var freq = {}; Object.values(byRow).forEach(function(cols) { var k = cols.sort().join(","); freq[k] = (freq[k]||0)+1; }); var entries = Object.entries(freq).sort(function(a,b){return b[1]-a[1];}); if(!entries.length) return "no seats"; var cols = entries[0][0].split(","); var n = cols.length; var aisle; if(n>=10) aisle=["C","D","G","H"]; else if(n===9) aisle=["C","D","F","G"]; else if(n===6) aisle=["C","D"]; else if(n===4) aisle=["D","G"]; else aisle=["C","D"]; return JSON.stringify({cols:n, aisle:aisle}); })()'
```

**佈局對照：**
| 艙位 | 列數 | 走廊字母 |
|------|------|---------|
| 經濟（Boeing 777, 3-4-3）| 10 | C、D、G、H |
| 經濟（787/A350, 3-3-3）| 9 | C、D、F、G |
| 經濟（A320/A321, 3-3）| 6 | C、D |
| 商務（1-2-1）| 4 | D、G |

#### 第二步：找最前排走廊位並點擊

根據偵測到的走廊字母，篩出所有可用走廊位，按排數由小至大，選第一個：

```bash
# 例：走廊字母 = ["C","D"]，找最前排可用走廊位
agent-browser --session-name trip eval '(function() { var doc = document.querySelector("iframe").contentDocument; var aisle = ["C","D"]; var seats = Array.from(doc.querySelectorAll("div.seat-item")).filter(function(el) { if(el.style.cursor !== "pointer") return false; var t = el.textContent.trim(); var m = t.match(/^(\d+)([A-Z]+)$/); return m && aisle.indexOf(m[2]) !== -1; }); seats.sort(function(a,b) { var ma = a.textContent.trim().match(/^(\d+)/); var mb = b.textContent.trim().match(/^(\d+)/); return parseInt(ma[1]) - parseInt(mb[1]); }); if(seats.length) { seats[0].click(); return "clicked " + seats[0].textContent.trim(); } return "no aisle seat found"; })()'
```

選座後：
- **來回程**：點「下一程航班」→ 選回程座位 → 點「確認」
- **單程**：直接點「確認」

```bash
# 來回程：去程選完後點下一程航班
agent-browser --session-name trip snapshot | grep -i "下一程航班\|確認"
agent-browser --session-name trip click {下一程航班 ref}
# 用相同方法選回程走廊位
# 點確認
agent-browser --session-name trip click {確認 ref}
```

告知用戶：
```
💺 去程座位：{SEAT}（走廊位，最前排）— HK$XXX
💺 回程座位：{SEAT}（走廊位，最前排）— HK$XXX
```

### 5g. Step 3：訂製行程（旅遊保險等）

座位選完後進入 Step 3，Trip.com 推銷保險等附加服務，直接點 `Next step >` 跳過：

```bash
agent-browser --session-name trip snapshot | grep -i "Next step"
agent-browser --session-name trip click {Next step ref}
```

### 5h. 完成後交由用戶（付款頁）

Step 3 完成後自動進入付款頁（Step 4/4）。截圖讓用戶確認：

```
✅ 訂票資料已全部填妥，請在瀏覽器視窗完成付款：

航班：CX251 HKG → LHR，直航
總額：HK$28,881（MasterCard 優惠 -HK$463）

請輸入信用卡 CVV/CVC 完成付款。
⚠️ AI 不會處理信用卡資料，付款由你自行操作。
```

---

## 首次使用：Trip.com 登入

如果 Trip.com session 未登入（右上角顯示「登入/註冊」）：

```bash
# 打開登入頁
agent-browser --session-name trip open "https://hk.trip.com/?locale=zh_hk&curr=HKD"
```

截圖後讓用戶在 headed browser 視窗完成登入。登入後 session 會持久保存 cookies，下次不需重新登入。

**確認已登入：**
```bash
# 已登入時 snapshot 會顯示用戶名稱按鈕，例如 button "Lau Chi Wang"
agent-browser --session-name trip snapshot | grep -E "button.*[A-Z][a-z]+ [A-Z][a-z]+|我的訂單"
```
若未見到用戶名稱，表示未登入，需讓用戶在 headed 視窗手動登入。

---

## 錯誤處理

| 情況 | 處理方法 |
|------|---------|
| 頁面仍在載入 | 再等 5 秒後重新截圖 |
| 未登入 Trip.com | 導航至 hk.trip.com 讓用戶登入 |
| 搜尋結果為空 | 建議調整日期或放寬條件重試 |
| 城市名稱未知 | 參考城市代碼對照表，或直接用 IATA 代碼 |
| snapshot 未見航班 | 等待更長時間或重新 open URL |

---

## 安全規則（不可違反）

1. ❌ **絕不代替用戶付款**
2. ❌ **絕不儲存信用卡資料**
3. ✅ **付款步驟永遠由用戶自行操作**
4. ✅ **只協助填寫乘客資料（非付款資料）**
