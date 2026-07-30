# 規格：帳戶頁重整（分類分組、順序調整、餘額正負號顯示、明細依時間排序）＋全站補建立時間戳

- 交接對象：Codex
- 交接人：Claude
- 對應背景：`02_Assistants/finance/Knowledge/DECISIONS.md` Decision 015 分工試驗，這是第三次交接
- **這份規格牽涉帳戶顯示邏輯，是 Decision 015 一開始刻意排除在試驗範圍外的區域，這次是 Wei 明確要求才納入。請嚴格照規格做，不要自行改動 `computeAccountBalance()` 的計算公式本身（只調整「顯示」這一層），也不要改動對帳頁（`tab-recon`）逐日累加/不配對的核心邏輯**
- 這份規格建立在前一份 `SPEC_收入支出頁面重整.md` 已經完成的基礎上（該份已經幫 `taxiLedgerExpenses` 加了 `createdAt`），這次要把同一套時間戳補齊到其他三種資料

---

## 1. 原始碼位置

- 唯一原始碼檔案：`/Users/weilee/Library/Mobile Documents/com~apple~CloudDocs/Document/AI資料庫/03_Projects/taxi-ledger-app/index.html`
- 需要一併修改：`/Users/weilee/Library/Mobile Documents/com~apple~CloudDocs/Document/AI資料庫/03_Projects/taxi-ledger-app/service-worker.js`（`CACHE_NAME` 版號同步 +1；如果 `SPEC_收入支出頁面重整.md` 那份已經做完並改成 `v25`，這次請改成 `v26`；如果那份還沒做，請先確認目前版號再 +1，不要跳號也不要撞號）
- 不要碰：`tab-recon` 對帳頁的**計算邏輯**（累加/不配對那套演算法本身不能動，只能加時間戳欄位）、`tab-today` 每日輸入的568/平台計算邏輯

## 2. 技術架構

- 單一 `index.html`，vanilla JS，`localStorage`
- 這次會動到的資料 key：`taxiLedgerAccounts`（帳戶）、`taxiLedgerIncomes`（收入明細）、`taxiLedgerTransfers`（轉帳）、`taxiLedgerBankDeposits`（銀行入帳/對帳，程式裡變數叫 `RECON_KEY`）

## 3. 已完成功能（跟這次改動相關，直接參考複用）

- **`computeAccountBalance(account)`**（約 1537 行）：已經有「信用帳戶 = 負債，消費/轉出增加負債、還款/收入減少負債」的完整換算邏輯，**這次不用改這個函式**，只是要把它算出來的 `balance` 在顯示時，比照這個函式已經在用的 type 判斷方式（信用帳戶顯示成負的），套用到 `renderAccounts()` 和 `renderAccountDetail()` 上
- **`accountDetailEditBtn`**（約 219 行）：「編輯帳戶」按鈕已經存在於帳戶明細頁右上角，點下去會關掉明細頁、把該帳戶資料載入下方的編輯表單。**這項不用做，已經有了，不要重複實作**
- **`renderAccountDetail()` 的 `items` 合併清單**（約 1691-1708 行）：已經把該帳戶的支出/收入/轉帳三種資料合併成一個陣列並依 `date` 排序，這次要把排序條件從純 `date` 擴充成 `date + createdAt`
- **`SPEC_收入支出頁面重整.md` 已經（或即將）幫 `taxiLedgerExpenses` 加上 `createdAt`**：新增時寫入 `new Date().toISOString()`、編輯時保留原值。這次對 `taxiLedgerIncomes`/`taxiLedgerTransfers`/`taxiLedgerBankDeposits` 三個資料表比照辦理，寫法完全一樣

## 4. 這次要做的功能（範圍界定）

- **要做**：
  1. 全站補齊建立時間戳：`taxiLedgerIncomes`、`taxiLedgerTransfers`、`taxiLedgerBankDeposits` 這三個資料表的新增流程都加上 `createdAt`
  2. 帳戶列表分成「現金」「信用卡」兩組，各自一個表頭，拿掉現在每個帳戶名稱後面重複顯示的類型文字
  3. 帳戶列表可以用「上移/下移」按鈕調整同類型帳戶之間的順序（原因見第7節）
  4. 帳戶列表/帳戶明細頁的餘額數字，信用帳戶要顯示成負數（沿用 `computeAccountBalance()` 已經內建的類型判斷，不新增欄位）
  5. 帳戶明細頁的收入/支出/轉帳合併清單，排序改成「日期+建立時間」新到舊
- **明確不做**：
  - 不做手指拖拉排序（這次先用上移/下移按鈕，原因見第7節，之後有需要可以再升級）
  - 不改 `computeAccountBalance()` 的計算公式本身，不改對帳頁的核心演算法
  - 不新增「手動調整正負號」的欄位，正負號完全由帳戶的 `type`（現金/信用）自動決定
  - 不動 `tab-today`（今日輸入）、`tab-expenses`、上一份規格已經處理的收入支出頁重整內容

## 5. 資料欄位與計算規則

- 三個資料表都新增 `createdAt`（ISO 字串）欄位，規則跟 `taxiLedgerExpenses` 一致：
  - 新增時：`createdAt: new Date().toISOString()`
  - 編輯既有紀錄時：保留原值，`createdAt: 原物件.createdAt || new Date().toISOString()`（後面這個保底只是給沒有時間戳的舊資料一個值，不是要覆蓋）
- **帳戶顯示用餘額**（不是新欄位，是既有 `computeAccountBalance()` 回傳值的顯示轉換）：
  ```js
  var displayBalance = (acc.type === 'credit') ? -balance : balance;
  ```
  這個轉換邏輯目前只在算「總餘額」時用過（約 1662 行 `a.type === 'credit' ? -b : b`），這次要在 `renderAccounts()` 的每一列、`renderAccountDetail()` 的餘額數字，都套用同一個轉換，維持三個地方（總餘額/列表/明細）的正負號規則一致
- **帳戶排序**：不新增 `order` 欄位，帳戶顯示順序就是 `taxiLedgerAccounts` 這個陣列本身的順序（本來就是這樣），上移/下移按鈕直接操作陣列順序、存回 `localStorage`
- **合併明細排序規則**：
  ```js
  items.sort(function (a, b) {
    if (a.date !== b.date) return b.date.localeCompare(a.date);
    return (b.createdAt || '').localeCompare(a.createdAt || '');
  });
  ```
  沒有 `createdAt` 的舊資料（這次上線前建立的）在同一天內排在最後面（視為當天最早建立）

## 6. 頁面流程與 UI 規格

### 6.1 三個資料表補建立時間戳

- `transferSaveBtn` click handler（約 1919 行）：
  ```js
  if (editingTransferId) {
    transfers = transfers.map(function (t) {
      return t.id === editingTransferId ? { id: t.id, date: date, fromAccountId: fromAccountId, toAccountId: toAccountId, amount: amount, note: note, createdAt: t.createdAt || new Date().toISOString() } : t;
    });
  } else {
    transfers.push({ id: uniqueId('t'), date: date, fromAccountId: fromAccountId, toAccountId: toAccountId, amount: amount, note: note, createdAt: new Date().toISOString() });
  }
  ```
- `incomeSaveBtn` click handler（約 2026 行），同樣寫法，`incomes.push({..., createdAt: new Date().toISOString()})`、編輯時 `createdAt: i.createdAt || new Date().toISOString()`
- `reconSaveBtn` click handler（約 1342 行），同樣寫法，`deposits.push({..., createdAt: new Date().toISOString()})`、編輯時 `createdAt: d.createdAt || new Date().toISOString()`
- 這三處都不影響原本的 `amount`/`date`/其他既有欄位，只新增這一個欄位，備份匯出/匯入不用改（整包搬資料，新欄位自動帶過去，已確認）

### 6.2 帳戶列表分組 + 上移/下移

`renderAccounts()`（約 1638 行）整個函式改寫邏輯：

```js
function renderAccounts() {
  var accounts = loadAccounts();
  var container = document.getElementById('accountList');
  container.innerHTML = '';
  if (!accounts.length) {
    container.innerHTML = '<p class="empty">還沒有帳戶,新增第一個吧</p>';
  } else {
    ['cash', 'credit'].forEach(function (type) {
      var group = accounts.filter(function (a) { return a.type === type; });
      if (!group.length) return;
      var header = document.createElement('div');
      header.className = 'account-group-header';
      header.textContent = ACCOUNT_TYPE_LABEL[type];
      container.appendChild(header);
      group.forEach(function (acc, idxInGroup) {
        var balance = computeAccountBalance(acc);
        var displayBalance = (acc.type === 'credit') ? -balance : balance;
        var currency = acc.currency || 'TWD';
        var row = document.createElement('div');
        row.className = 'account-row' + (acc.archived ? ' archived' : '');
        row.innerHTML =
          '<span class="account-reorder">' +
            '<button type="button" class="account-move-btn" data-dir="up" data-id="' + acc.id + '"' + (idxInGroup === 0 ? ' style="visibility:hidden;"' : '') + '>▲</button>' +
            '<button type="button" class="account-move-btn" data-dir="down" data-id="' + acc.id + '"' + (idxInGroup === group.length - 1 ? ' style="visibility:hidden;"' : '') + '>▼</button>' +
          '</span>' +
          '<span class="account-name-wrap"><span class="account-name">' + acc.name + '</span>' + (currency !== 'TWD' ? '<span class="account-type-badge">' + currency + '</span>' : '') + (acc.archived ? '<span class="account-type-badge">已封存</span>' : '') + '</span>' +
          '<span class="account-balance tnum ' + (displayBalance < 0 ? 'neg' : 'pos') + '">' + (currency !== 'TWD' ? currency + ' ' : '') + fmt(displayBalance) + '</span>';
        row.querySelectorAll('.account-move-btn').forEach(function (btn) {
          btn.addEventListener('click', function (e) {
            e.stopPropagation();
            moveAccount(btn.getAttribute('data-id'), btn.getAttribute('data-dir'));
          });
        });
        row.addEventListener('click', function () { showAccountDetail(acc.id); });
        container.appendChild(row);
      });
    });
  }
  var total = accounts.filter(function (a) { return a.includeInTotal !== false && !a.archived && (a.currency || 'TWD') === 'TWD'; })
    .reduce(function (s, a) { var b = computeAccountBalance(a); return s + (a.type === 'credit' ? -b : b); }, 0);
  document.getElementById('accountsTotalBalance').textContent = fmt(total);
}

function moveAccount(id, dir) {
  var accounts = loadAccounts();
  var acc = accounts.find(function (a) { return a.id === id; });
  if (!acc) return;
  var sameTypeIndexes = accounts.map(function (a, i) { return a.type === acc.type ? i : -1; }).filter(function (i) { return i !== -1; });
  var posInType = sameTypeIndexes.indexOf(accounts.indexOf(acc));
  var swapWithPos = dir === 'up' ? posInType - 1 : posInType + 1;
  if (swapWithPos < 0 || swapWithPos >= sameTypeIndexes.length) return;
  var i1 = accounts.indexOf(acc), i2 = sameTypeIndexes[swapWithPos];
  var tmp = accounts[i1]; accounts[i1] = accounts[i2]; accounts[i2] = tmp;
  saveAccountsList(accounts);
  renderAccounts();
}
```

（`ACCOUNT_TYPE_LABEL` 已經存在，`{cash:'現金', credit:'信用'}`，這裡拿它當表頭文字。如果 Wei 之後想把「信用」表頭文字改成「信用卡」三個字，可以直接改這個常數，不影響邏輯）

CSS 新增：
```css
.account-group-header { font-size: 13px; font-weight: 700; color: var(--ink-muted); padding: 10px 4px 4px; text-transform: uppercase; letter-spacing: .04em; }
.account-reorder { display: flex; flex-direction: column; gap: 2px; margin-right: 8px; }
.account-move-btn { width: 24px; height: 24px; border: none; background: var(--surface-alt); border-radius: 6px; color: var(--ink-muted); font-size: 11px; cursor: pointer; padding: 0; line-height: 24px; }
.account-name-wrap { flex: 1; min-width: 0; }
```
`.account-row` 原本是 `display:flex; align-items:center; justify-content:space-between;`（約 59 行），這次三個子元素（reorder / name / balance）並排即可，不用改這條既有規則。

**重要**：上移/下移按鈕點擊要 `e.stopPropagation()`，避免誤觸發整列的 `showAccountDetail()`（這是唯一一個容易漏掉、一漏就會出現「點上移結果跳去帳戶明細」的bug，請務必確認測試過）。

### 6.3 帳戶明細頁餘額顯示（`renderAccountDetail()`，約 1676 行）

原本：
```js
var balance = computeAccountBalance(acc);
document.getElementById('accountDetailName').textContent = acc.name;
var balEl = document.getElementById('accountDetailBalance');
balEl.textContent = (currency !== 'TWD' ? currency + ' ' : '') + fmt(balance);
balEl.className = 'v tnum ' + ((acc.type === 'credit' ? -balance : balance) < 0 ? 'neg' : 'pos');
```
改成：
```js
var balance = computeAccountBalance(acc);
var displayBalance = (acc.type === 'credit') ? -balance : balance;
document.getElementById('accountDetailName').textContent = acc.name;
var balEl = document.getElementById('accountDetailBalance');
balEl.textContent = (currency !== 'TWD' ? currency + ' ' : '') + fmt(displayBalance);
balEl.className = 'v tnum ' + (displayBalance < 0 ? 'neg' : 'pos');
```
（只是把原本重複寫兩次的 `acc.type === 'credit' ? -balance : balance` 收斂成一個變數，數字現在也套用同一個轉換，行為上就是信用帳戶會顯示負數）

### 6.4 帳戶明細合併清單排序

`renderAccountDetail()` 裡組 `items` 的地方（約 1692-1707 行），三個 `items.push(...)` 都加上 `createdAt: e.createdAt`（或 `i.createdAt`/`t.createdAt`），然後把排序那行（約 1708 行）：
```js
items.sort(function (a, b) { return b.date.localeCompare(a.date); });
```
改成：
```js
items.sort(function (a, b) {
  if (a.date !== b.date) return b.date.localeCompare(a.date);
  return (b.createdAt || '').localeCompare(a.createdAt || '');
});
```

## 7. 已確認決策及原因

- **這次先做「上移/下移按鈕」，不做手指拖拉排序**：Wei 原本期待的是手指拖拉（比較符合手機記帳App的直覺），但把最終判斷交給 Claude 決定。Claude 的判斷是：這是單檔 vanilla JS、沒有任何前端框架或手勢函式庫的專案，要做順暢、不誤觸、跟頁面本身捲動不衝突的觸控拖拉，需要自己刻手勢判斷邏輯，複雜度和出錯風險都明顯比上移/下移按鈕高很多；而這次同時又是分工試驗規模第一次擴大到「牽涉帳戶顯示」的範圍，風險已經比前兩次高，不適合再疊加一個高複雜度的手勢功能。上移/下移按鈕能達到一樣的「調整順序」效果，之後如果這幾批規格都驗證順利、Wei 還是想要更順手的拖拉手感，可以再另外排一次規格單獨做
- **帳戶排序不新增 `order` 欄位，直接用陣列順序**：帳戶數量通常不多（現金+信用卡各自可能就幾個），不需要為了排序另外設計欄位，直接操作既有陣列並存回去最簡單，跟現有「陣列順序=顯示順序」的既有慣例一致（例如 `taxiLedgerDays`、支出等資料表都沒有額外的排序欄位，都是即時 sort 或原始陣列順序）
- **餘額正負號完全由帳戶類型自動決定，不開放手動調整**：Wei 確認過，因為 App 裡「信用帳戶=負債」這套換算邏輯已經是既有、驗證過的規則（`computeAccountBalance()`），如果額外開一個「手動調整正負號」的欄位，等於在既有規則之外疊加一層人工設定，兩套邏輯之後要對齊、互相打架的風險增加，這次只是把「總餘額」已經在用的判斷方式，同步套到「列表」和「明細」的顯示上，維持三處一致
- **這次跨足了對帳資料結構（`taxiLedgerBankDeposits`）加時間戳**：Decision 015 一開始說好這批試驗不牽涉對帳，但 Wei 明確要求「所有新增的紀錄都要有日期跟時間，不管哪一種類型」，所以把這個欄位新增也涵蓋進對帳資料。**這只新增一個不影響任何金額計算的 metadata 欄位，完全沒有動到對帳頁「逐日累加、不配對」的核心演算法本身**，是在 Wei 明確指示下有意識地小範圍跨界，不是自行擴大範圍

## 8. 下一個開發任務（不在這次範圍內，僅供參考）

- 如果之後想把「上移/下移」升級成手指拖拉排序，需要另外排一次規格，重點會是：長按觸發、跟頁面垂直捲動手勢的衝突處理、拖拉中的視覺回饋（佔位陰影等）
- 貸款帳戶（Decision 006 提過，Wei 明確要求先不做）：這次的「現金/信用卡」分組沒有把貸款帳戶算進去，之後如果要做貸款帳戶，分組邏輯要記得擴充第三組

## 9. 驗收條件

- [ ] 新增一筆收入/轉帳/銀行入帳，三種資料在 `localStorage` 裡都看得到 `createdAt` 欄位；編輯既有紀錄後 `createdAt` 維持原值不變
- [ ] 帳戶頁看到帳戶列表分成「現金」「信用卡」兩個表頭群組，現金在上面
- [ ] 每個帳戶名稱後面不再重複顯示類型文字（只有幣別/已封存這種額外標記還在）
- [ ] 同類型底下至少兩個帳戶時，點上移/下移按鈕，順序正確調整，重新整理頁面後順序還在（有存進 `localStorage`）
- [ ] 每個群組最上面的帳戶看不到「上移」按鈕（或按鈕看得到但點了沒反應/隱藏起來）、最下面的帳戶看不到「下移」按鈕
- [ ] 點上移/下移按鈕，**不會**連帶跳出帳戶明細頁（確認有正確 `stopPropagation`）
- [ ] 新增一個信用帳戶、給一個正的初始金額（代表欠款），帳戶列表跟帳戶明細頁的餘額數字都顯示成負數（不是只有顏色是紅/橘色，數字本身要有負號），現金帳戶維持顯示正數
- [ ] 「總餘額」的計算結果跟改版前完全一樣（這次只改「單一帳戶」的顯示，不能連帶把總餘額算錯）
- [ ] 帳戶明細頁裡，同一天有多筆支出+收入+轉帳混合時，越晚建立的排越上面
- [ ] `service-worker.js` 的 `CACHE_NAME` 版號有跟著 +1
- [ ] 用 `git diff` 確認沒有改到 `computeAccountBalance()` 的計算公式本身、沒有改到對帳頁的累加/比對演算法

## 10. 目前哪些檔案正在被誰改（避免衝突）

- 這次任務期間，只有 Codex 會改 `index.html` 和 `service-worker.js`
- 如果 `SPEC_收入支出頁面重整.md` 那份還沒做完，這兩份規格請不要同時派給兩個不同的 Codex 對話/視窗同時改同一個檔案，避免衝突；建議照順序做完一份、驗收過、再做下一份
- 完成後請自行 commit（訊息建議照現有慣例，例如「帳戶頁重整:分類分組/上下移排序/餘額負數顯示/明細依時間排序+全站補建立時間戳(vXX)」），**不要 push**，push 前要先讓 Wei 確認
- 完成後回報：驗收條件逐項有沒有過、有沒有發現規格裡沒講清楚或漏掉的地方
