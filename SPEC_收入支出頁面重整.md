# 規格：收入支出頁面重整(改名、568拆格、統計按鍵放大、支出加時間戳並併入歷史紀錄)

- 交接對象：Codex
- 交接人：Claude
- 對應背景：`02_Assistants/finance/Knowledge/DECISIONS.md` Decision 015 分工試驗，這是第二次交接（第一次「休假日視覺標記」已驗收通過）
- **這份規格只涵蓋 UI/顯示/排序調整，明確不牽涉帳戶餘額計算、對帳、CSV。帳戶相關的改動（分類分組、拖拉排序、餘額正負號）是另一份獨立規格，這次不要動帳戶頁（`tab-accounts`）任何程式碼**

---

## 1. 原始碼位置

- 唯一原始碼檔案：`/Users/weilee/Library/Mobile Documents/com~apple~CloudDocs/Document/AI資料庫/03_Projects/taxi-ledger-app/index.html`
- 需要一併修改：`/Users/weilee/Library/Mobile Documents/com~apple~CloudDocs/Document/AI資料庫/03_Projects/taxi-ledger-app/service-worker.js`（`CACHE_NAME` 版號同步 +1，目前是 `taxi-ledger-v24`，這次改完請改成 `taxi-ledger-v25`）
- 不需要碰：`manifest.json`、`icons/`、`tab-accounts`（帳戶頁）、`tab-recon`（對帳頁）、`tab-incomes`（收入明細頁，跟這次的「收入」子分頁是不同的資料表，不要動）

## 2. 技術架構

- 單一 `index.html`，vanilla JS，`localStorage` 持久化，PWA（`service-worker.js` 快取，改完 `index.html` 記得同步升版號，這是這個專案的既有慣例）
- 相關資料 key：`taxiLedgerDays`（每日 568/平台收入，`STORAGE_KEY`）、`taxiLedgerExpenses`（逐筆支出，`EXPENSES_KEY`）

## 3. 已完成功能（跟這次改動相關、可以直接參考複用）

- **`.seg-group`/`.seg-btn`**（約 159-161 行 CSS）：現有的分頁式切換按鈕樣式（例如統計頁「天/週/月/年」），這次「收入/支出」子分頁請直接沿用同一組 class，不要另外設計新樣式
- **`renderRecentList()`**（約 965 行）：目前渲染「歷史紀錄」卡片裡「最近7天」的收入清單，抓 `loadDays()`裡最近7天的估算總額，點擊某一列會呼叫 `loadDateIntoForm(date)` 跳去編輯那天。這次改動後這個函式的邏輯不變，只是外層要包一層「收入」子分頁的顯示切換
- **`renderExpenseList()`**（約 1842 行）：目前渲染 `tab-expenses` 裡「支出紀錄」卡片的清單，抓全部 `taxiLedgerExpenses`、依 `date` 字串排序、點擊某一列會呼叫 `loadExpenseIntoForm(exp)` 並跳回表單編輯。這次要把這個函式**拆成共用的列渲染邏輯**，供新的兩個支出清單位置共用（見第6節）
- **`renderCalendar()`**（約 917 行）：目前是「歷史紀錄」區塊唯一的中央刷新點，內部已經呼叫 `renderRecentList()`；已知所有會改到 `editingDate` 或存檔的地方（`loadDateIntoForm`、`persistCurrentEntry`、`backToToday`、`deleteDayBtn`、切到今日輸入分頁、匯入備份）最後都會呼叫 `renderCalendar()`。**這次新增的渲染函式請比照現有模式，一併掛在 `renderCalendar()` 裡面**，不要在每個呼叫點個別加，這樣可以確保所有現有的刷新時機都會自動涵蓋到，不用擔心漏掉某個入口
- **`accountDetailEditBtn`**：這是屬於帳戶頁的功能，跟這份規格無關，只是說明「編輯帳戶」按鈕本來就存在，不在這次任務範圍內，提醒不要因為看到相關程式碼而誤觸

## 4. 這次要做的功能（範圍界定）

- **要做**：
  1. 「營業收入」分頁改名為「收入支出」
  2. 「568 平台派單」卡片裡的「預估銀行應入帳」拆成兩格：「568 總入帳」+「預估銀行入帳」
  3. 「統計」分頁「收入總覽」卡片的上一期/下一期按鍵放大（目前視覺上看起來大，但實際可點擊範圍只有 20x20px，是這次要修的真正問題）
  4. 支出紀錄加上建立時間戳，清單依「日期+建立時間」新到舊排序
  5. 「支出紀錄」清單搬出 `tab-expenses`，改成顯示在「收入支出」分頁「歷史紀錄」卡片裡，跟收入並列成「收入/支出」兩個子分頁：一個是「最近7天」層級（沿用現有 `renderRecentList()` 邏輯），一個是「點進某一天」層級（顯示 `editingDate` 那天的收入明細+支出明細）
- **明確不做**（超出這次範圍，不要順手做）：
  - 不碰帳戶頁（`tab-accounts`）：分類分組、拖拉排序、編輯功能、餘額正負號顯示，都是另一份規格的事
  - 不碰對帳頁（`tab-recon`）、不碰 CSV 匯入
  - 不碰 `tab-incomes`（收入明細，逐筆的其他收入紀錄）：這次的「收入」子分頁顯示的是「今日輸入」的568/平台每日收入（`taxiLedgerDays`），跟 `tab-incomes` 是完全不同的兩份資料，確認過不要混在一起
  - 不處理 iOS 原生鍵盤上方「上一欄/下一欄/完成」工具列——那是 Mobile Safari 對數字鍵盤欄位自動加的原生 UI，程式碼裡找不到對應元素，web 內容沒有 API 可以關閉它，這次不列入任務

## 5. 資料欄位與計算規則

- **568 總入帳**：`cash568 + online568`（就是「568 現金」+「568 綁定」兩個既有欄位的原始加總，不含 Uber/Bolt/yoxi/直客/其他），沒有新計算規則，只是把已經存在的兩個數字相加後另外顯示
- **預估銀行入帳**：維持現有 `calcNetOnline568(cash, online)` 計算邏輯完全不變（`online*0.85 - cash*0.15`），只是搬到新的兩格版面裡，數字跟色彩規則（`neg`/`pos`）都不變
- **支出建立時間戳**：新增欄位 `createdAt`（ISO 字串，`new Date().toISOString()`）
  - 新增支出時：`createdAt = new Date().toISOString()`
  - 編輯既有支出時：**保留原本的 `createdAt`，不要更新**（`e.createdAt || new Date().toISOString()`，後者只是給沒有這個欄位的舊資料一個保底值，避免編輯舊資料時掉欄位）
  - 沒有任何欄位刪除或改變既有 `date`/`name`/`amount`/`category`/`accountId` 的意義，`createdAt` 純粹是新增欄位，不影響 `computeAccountBalance()` 或任何金額加總
- **支出排序規則**：同一天有多筆支出時，`createdAt` 新的排上面；沒有 `createdAt` 的舊資料（這次上線前建立的）視為最早，排在同一天所有有時間戳的紀錄後面。跨天的排序維持「日期新的在上面」優先於時間戳

## 6. 頁面流程與 UI 規格

### 6.1 分頁改名

把以下 4 處使用者看得到的「營業收入」字樣改成「收入支出」：
- 第 518 行 `<button ... data-tab="today">營業收入</button>` → `收入支出`
- 第 210 行 `<p class="sub">營業收入 · 資料只存在這支裝置</p>` → `收入支出 · 資料只存在這支裝置`
- 第 6 行 `<title>記帳本 · 營業收入</title>` → `記帳本 · 收入支出`
- 第 505 行 `settings-hint` 裡「跟「營業收入」分頁的568/Uber每日計算機」→「跟「收入支出」分頁的568/Uber每日計算機」
- 第 510 行 foot 文字「記帳本 v3 · 營業收入 / 統計 / 對帳 / 帳戶 / 支出」→「記帳本 v3 · 收入支出 / 統計 / 對帳 / 帳戶」（結尾「/ 支出」拿掉，因為支出本來就不是獨立分頁，這次更明確整合進「收入支出」的歷史紀錄裡）
- `data-tab="today"` 這個屬性值本身不要改，只改按鈕顯示文字，JS 裡引用 `'today'` 的地方都不用動

### 6.2 568 總入帳/預估銀行入帳 拆兩格

HTML（約 284-287 行），原本：
```html
<div class="hero">
  <div class="k">預估銀行應入帳</div>
  <div class="v tnum" id="netOnline568">$0</div>
</div>
```
改成：
```html
<div class="hero-row">
  <div class="hero" style="flex:1;">
    <div class="k">568 總入帳</div>
    <div class="v tnum" id="total568">$0</div>
  </div>
  <div class="hero" style="flex:1;">
    <div class="k">預估銀行入帳</div>
    <div class="v tnum" id="netOnline568">$0</div>
  </div>
</div>
```
CSS 新增：
```css
.hero-row { display: flex; gap: 8px; }
```
JS：`updateLiveCalc()`（約 716 行）裡，`var cash = num('cash568'), online = num('online568');` 之後加一行：
```js
document.getElementById('total568').textContent = fmt(cash + online);
```
`total568` 不需要 `neg`/`pos` 顏色判斷（568現金+綁定一定是正數或0，不會是負的），維持預設樣式即可，不用加 class。

### 6.3 統計頁上一期/下一期按鍵放大

CSS（約 178 行），原本：
```css
.cal-nav.small { width: 20px; height: 20px; overflow: visible; font-size: 44px; line-height: 1; background: var(--accent); color: var(--accent-ink); border-color: var(--accent); }
```
改成：
```css
.cal-nav.small { width: 44px; height: 44px; overflow: visible; font-size: 24px; line-height: 1; background: var(--accent); color: var(--accent-ink); border-color: var(--accent); border-radius: 10px; }
```
原因：舊寫法用 `overflow:visible` + 超大字級讓箭頭「看起來」大，但實際可點擊的框只有 20x20px，手指常點不到。改成真正 44x44px 的框（符合 Apple 建議的最小點擊尺寸），字級縮小到跟框比例相稱，讓它看起來是一個清楚的「按鈕格子」，不是浮空的大符號。這個 class 目前只有 `statsPrevBtn`/`statsNextBtn` 在用（已確認，對帳頁/月曆頁用的是別的 class），改了不會影響其他地方。

### 6.4 支出搬進「歷史紀錄」+ 收入/支出子分頁

**這是這次規格最大的一塊，請按小節逐步做：**

#### (a) 拿掉 `tab-expenses` 裡的「支出紀錄」清單卡片

刪掉這個區塊（約 483-486 行）：
```html
<div class="card">
  <h2>支出紀錄</h2>
  <div id="expenseList"><p class="empty">還沒有記錄</p></div>
</div>
```
`tab-expenses` 之後只留「新增支出」表單卡片（約 470-481 行），不要留清單。

#### (b) 把 `renderExpenseList()` 拆成共用的列渲染函式

新增一個共用函式，接收「要渲染的支出陣列」和「目標容器」，回傳/插入渲染好的列（沿用原本 `renderExpenseList()` 內部組 HTML 的邏輯、包含 `sub` 欄位跟點擊跳轉編輯的行為）：
```js
function renderExpenseRows(container, expenses) {
  var labels = loadCategoryLabels();
  var accounts = loadAccounts();
  var accountName = {};
  accounts.forEach(function (a) { accountName[a.id] = a.name; });
  container.innerHTML = '';
  if (!expenses.length) { container.innerHTML = '<p class="empty">還沒有支出紀錄</p>'; return; }
  expenses.forEach(function (exp) {
    var sub = (labels[exp.category] || exp.category) + (exp.accountId && accountName[exp.accountId] ? ' · ' + accountName[exp.accountId] : '');
    var timeLabel = exp.createdAt ? (' ' + formatTimeLabel(exp.createdAt)) : '';
    var row = document.createElement('div');
    row.className = 'recent-item';
    row.innerHTML =
      '<span class="recent-date tnum">' + formatDateLabel(exp.date) + timeLabel + ' ' + exp.name + '<br><span style="font-size:12px; color:var(--ink-muted); font-weight:400;">' + sub + '</span></span>' +
      '<span class="recent-amt tnum">' + fmt(exp.amount) + '</span>';
    row.addEventListener('click', function () {
      switchTab('expenses');
      loadExpenseIntoForm(exp);
      window.scrollTo({ top: 0, behavior: 'smooth' });
    });
    container.appendChild(row);
  });
}

function formatTimeLabel(iso) {
  var d = new Date(iso);
  return pad2(d.getHours()) + ':' + pad2(d.getMinutes());
}

function compareExpenseDesc(a, b) {
  if (a.date !== b.date) return b.date.localeCompare(a.date);
  return (b.createdAt || '').localeCompare(a.createdAt || '');
}
```
（`pad2` 已經存在於現有程式碼裡，直接複用，不用重新定義）

刪掉舊的 `renderExpenseList()` 函式本體，改成呼叫 `renderExpenseRows()` 的地方見下面 (c)(d)。

#### (c) 「歷史紀錄」卡片：最近7天，收入/支出子分頁

HTML（約 315-317 行），原本：
```html
<h2 style="margin-top:18px;">最近 7 天</h2>
<div id="recentList"></div>
```
改成：
```html
<h2 style="margin-top:18px;">最近 7 天</h2>
<div class="seg-group" id="recentTabGroup">
  <button type="button" class="seg-btn active" data-subtab="income">收入</button>
  <button type="button" class="seg-btn" data-subtab="expense">支出</button>
</div>
<div id="recentList"></div>
<div id="recentExpenseList" style="display:none"></div>
```
JS：
- 新增 `recentTabGroup` 的點擊處理，切換 `#recentList`/`#recentExpenseList` 的 `display`，並切換 `.seg-btn.active`（照抄現有 `statsGranGroup` 的 `.seg-btn` 切換寫法）
- 新增函式 `renderRecentExpenseList()`：取最近 7 天（跟 `renderRecentList()` 同一個日期範圍算法：從今天往回數 6 天）的 `loadExpenses()`，用 `compareExpenseDesc` 排序後丟給 `renderExpenseRows(document.getElementById('recentExpenseList'), 排序後的陣列)`
- 在 `renderCalendar()` 裡，緊接在現有 `renderRecentList();` 那行後面加一行 `renderRecentExpenseList();`（讓它跟現有收入清單一樣，自動掛在既有的中央刷新點上，見第3節說明）

#### (d) 「歷史紀錄」卡片：點進某一天，收入/支出子分頁

HTML：在「有紀錄的日期」圖例（`.cal-legend`，約 314 行）之後、「最近7天」`<h2>` 之前，插入：
```html
<div class="daydetail-block">
  <h2 id="dayDetailTitle">今天的收支</h2>
  <div class="seg-group" id="dayDetailTabGroup">
    <button type="button" class="seg-btn active" data-subtab="income">收入</button>
    <button type="button" class="seg-btn" data-subtab="expense">支出</button>
  </div>
  <div id="dayDetailIncome"></div>
  <div id="dayDetailExpense" style="display:none"></div>
</div>
```
JS：
- `dayDetailTabGroup` 切換邏輯同 (c)
- 新增函式 `updateDayDetailTitle()`：`editingDate === todayKey()` 時標題是「今天的收支」，否則是 `formatDateLabel(editingDate) + ' 的收支'`
- 新增函式 `renderDayDetailIncome()`：讀 `loadDays()` 裡 `date === editingDate` 的那筆 entry（沒有的話顯示 `'<p class="empty">這天還沒有輸入收入</p>'`）；有的話，用現有的 `ids`（`cash568/online568/uber/bolt/yoxi/direct/other`）搭配 `loadLabels()` 取得的欄位名稱，逐項列出「名稱：金額」（金額是 0 的欄位也列出來，不要濾掉，維持完整對照），最後加一行總額（用現有 `updateLiveCalc()` 同一套 `estimated` 算法：`cash + calcNetOnline568(cash,online) + 其他平台加總`，或直接複用 `todayEstimated` 當下已經算好的值也可以，只要顯示的數字跟「今日輸入」表單上方的「今日/該日預估收入」一致即可）
- 新增函式 `renderDayDetailExpense()`：`loadExpenses().filter(function(e){ return e.date === editingDate; })`，用 `compareExpenseDesc` 排序後丟給 `renderExpenseRows(document.getElementById('dayDetailExpense'), 排序後的陣列)`
- 在 `loadDateIntoForm(date)`（約 860 行）裡，跟現有 `updateDayOffButtonState();`、`renderCalendar();` 放在一起的地方，加上呼叫 `updateDayDetailTitle(); renderDayDetailIncome(); renderDayDetailExpense();`
- 這三個函式也要掛進 `renderCalendar()`（同第3節說的中央刷新點模式），確保存檔/刪除支出後這個區塊會跟著刷新

#### (e) 支出表單儲存/刪除邏輯要加上 `createdAt` 並改用新的刷新方式

`expenseSaveBtn` click handler（約 1813 行）：
```js
if (editingExpenseId) {
  expenses = expenses.map(function (e) {
    return e.id === editingExpenseId ? { id: e.id, date: date, name: name, amount: amount, category: category, accountId: accountId, createdAt: e.createdAt || new Date().toISOString() } : e;
  });
} else {
  expenses.push({ id: uniqueId('e'), date: date, name: name, amount: amount, category: category, accountId: accountId, createdAt: new Date().toISOString() });
}
saveExpensesList(expenses);
resetExpenseForm();
renderCalendar(); // 原本是 renderExpenseList()，因為清單搬到歷史紀錄底下了，改成呼叫 renderCalendar() 讓新的兩個支出清單一起刷新
```
`expenseDeleteBtn` click handler（約 1833 行）同樣把 `renderExpenseList();` 改成 `renderCalendar();`

#### (f) quick-add「🧾 加一筆支出」流程不變

`quickAddExpenseBtn` 目前是 `switchTab('expenses')`，跳到只剩表單的 `tab-expenses`，行為不用改，使用者存完後可以自己滑回「收入支出」分頁的歷史紀錄看到剛存的那筆。

## 7. 已確認決策及原因

- **「支出」清單完全從 `tab-expenses` 移除，不是複製一份**：避免同一份資料在兩個地方各自渲染、之後容易漏改其中一處造成不一致；`tab-expenses` 之後只保留「新增/編輯」表單的角色
- **「收入」子分頁資料來源是 `taxiLedgerDays`（今日輸入的568/平台收入），不是 `taxiLedgerIncomes`（收入明細）**：這是 Wei 確認過的，因為使用者日常認知的「收入」就是每天在「今日輸入」打的數字，`taxiLedgerIncomes` 是另一個做個案請款/歷史回填用的獨立小工具，兩者故意不合併，避免混淆
- **「這天的收支」區塊即使選到「今天」也會顯示，不是只有選別天才出現**：維持行為一致（不管點哪天，同一個區塊固定顯示「目前正在編輯的那天」），比「今天特別隱藏」的例外邏輯簡單、不容易出錯
- **`createdAt` 只在新增時寫入、編輯時保留原值**：符合「建立當下的時間」這個字面意思，如果編輯時也更新，時間戳的意義會變成「最後修改時間」，跟 Wei 的原始需求不符
- **統計頁按鍵改成真正 44x44px 的框，不是繼續放大字級**：舊寫法字級看起來大但可點擊範圍沒變，是典型的「視覺大小」跟「可互動範圍」沒對齊的手機 UX 問題，這次直接修可點擊範圍本身

## 8. 下一個開發任務（不在這次範圍內，僅供參考）

- 帳戶頁重整（分類分組現金/信用卡、拖拉排序、編輯功能位置、餘額正負號顯示、帳戶明細依日期時間排序）：這是 Decision 015 刻意排除在這次之外的「帳戶核心邏輯」區域，Claude 會先設計過、跟 Wei 確認欄位邏輯後才會另外交接，不要提前動 `tab-accounts` 的程式碼
- 附帶一提：帳戶明細要做到「依日期時間排序」，最終需要 `taxiLedgerIncomes`/`taxiLedgerTransfers`/`taxiLedgerBankDeposits568` 也都補上建立時間戳（目前只有這次幫 `taxiLedgerExpenses` 加），這個缺口會在帳戶那份規格裡一併處理，這次不用管

## 9. 驗收條件

- [ ] 「今日輸入」分頁按鈕文字變成「收入支出」，其餘 3 處「營業收入」文字（title、副標題、hint 說明）都改成「收入支出」，foot 文字結尾的「/ 支出」拿掉
- [ ] 568 平台派單卡片變成兩格並排：「568 總入帳」顯示 cash568+online568 的加總、「預估銀行入帳」數字跟顏色規則跟改版前的「預估銀行應入帳」完全一致
- [ ] 統計頁「收入總覽」的上一期/下一期按鍵，實際可點擊範圍明顯變大（不是只有視覺變大），點擊行為（日/週/月各自的跳動邏輯）不受影響
- [ ] `tab-expenses` 只剩「新增支出」表單，沒有「支出紀錄」清單卡片
- [ ] 「歷史紀錄」卡片裡看得到兩組「收入/支出」子分頁：一組在「最近7天」下面、一組在新的「今天的收支」區塊裡
- [ ] 新增一筆支出後，「最近7天」的「支出」子分頁跟「今天的收支」的「支出」子分頁都會立刻看到這筆（不用重新整理）
- [ ] 同一天新增兩筆支出，清單裡後存的排在前存的上面；有時間戳的顯示格式類似「7/30 14:32 品名」
- [ ] 點月曆上某一天，「今天的收支」標題變成那天的日期、收入/支出子分頁內容也正確換成那一天的資料，不會殘留前一天的內容
- [ ] 「收入」子分頁顯示的收入資料，數字要跟「今日輸入」表單當下輸入的568/平台欄位一致（含 0 元欄位也列出來）
- [ ] 點支出清單裡任一筆，會跳回「新增支出」表單並帶入該筆資料可編輯，儲存後金額、分類、帳戶、`createdAt` 都不跑掉（`createdAt` 保留原值不變）
- [ ] `service-worker.js` 的 `CACHE_NAME` 已經改成 `taxi-ledger-v25`
- [ ] 用 `git diff` 確認沒有改到 `tab-accounts`、`tab-recon`、`tab-incomes`、`computeAccountBalance()` 相關程式碼

## 10. 目前哪些檔案正在被誰改（避免衝突）

- 這次任務期間，只有 Codex 會改 `index.html` 和 `service-worker.js`，Claude／Wei 不會同時改這兩個檔案
- 完成後請自行 commit（訊息建議照現有 git log 慣例，例如「收入支出頁面重整:改名/568拆格/統計按鍵放大/支出加時間戳併入歷史紀錄(v25)」），**不要 push**，push 前要先讓 Wei 確認
- 完成後回報：驗收條件逐項有沒有過、有沒有發現規格裡沒講清楚或漏掉的地方（尤其是 (c)(d) 這兩段結構比較複雜，如果哪裡卡住請具體說是卡在哪個函式/哪個 UI 狀態，方便下次補規格時參考）
