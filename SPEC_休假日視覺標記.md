# 規格：休假日視覺標記

- 交接對象：Codex
- 交接人：Claude
- 對應背景：`02_Assistants/finance/Knowledge/DECISIONS.md` Decision 015（Claude 設計架構＋Codex 執行製作分工試驗）與 Decision 007 第 6 項
- 這是試驗性交接的第一個功能，目的是驗證「規格寫多細，Codex 才不用回頭問」，請照規格逐項實作，不要自行擴充範圍或補充規格沒提到的細節

---

## 1. 原始碼位置

- 唯一原始碼檔案：`/Users/weilee/Library/Mobile Documents/com~apple~CloudDocs/Document/AI資料庫/03_Projects/taxi-ledger-app/index.html`
- 需要一併修改：`/Users/weilee/Library/Mobile Documents/com~apple~CloudDocs/Document/AI資料庫/03_Projects/taxi-ledger-app/service-worker.js`（原因見第 6 節最後一項）
- 不需要碰：`manifest.json`、`icons/`

## 2. 技術架構

- 單一 `index.html`，vanilla JS（無框架、無建置流程、無 npm），`<style>` 內嵌 CSS，`<script>` 內嵌 JS，全部寫在同一個檔案裡
- 資料存在瀏覽器 `localStorage`，逐日紀錄存在 key `taxiLedgerDays`（`STORAGE_KEY` 變數），是一個陣列，每個元素是一天的 entry 物件
- 是 PWA，`service-worker.js` 用 network-first + cache fallback 策略快取靜態檔案，`CACHE_NAME` 是手動版號（目前 `taxi-ledger-v23`）

## 3. 已完成功能（跟這次改動相關、可以直接參考複用的既有機制）

- **打卡上下班按鈕**（`punchBtn`，約在 index.html 815 行附近）：已經是「點一下＝寫入現在時間、自動存檔、震動回饋」的單鍵切換模式，樣式用 `.btn.punch-active` 表示啟用中的狀態。這次新功能請直接複製這個互動模式（點擊即存檔，不用等使用者按「儲存」鍵）
- **歷史紀錄月曆**（`renderCalendar()` 函式，約 889 行）：目前月曆格子（`.cal-cell`）只用一個小圓點（`.has-record::after`）標示「這天有紀錄」，還沒有其他視覺分類
- **每日 entry 資料物件**（`persistCurrentEntry()`，約 1033 行）：目前欄位有 `date`、`savedAt`、`cash568`、`online568`、`uber`、`bolt`、`yoxi`、`direct`、`other`、`startTime`、`endTime`
- **資料備份匯出/匯入**（約 2045-2131 行）：是整包搬 `taxiLedgerDays` 陣列，不挑欄位，所以 entry 物件上新增的欄位會自動被匯出/匯入，**這部分不用改**

## 4. 這次要做的功能（範圍界定）

- **要做**：在「今日輸入」頁加一個「休假」切換按鈕，可以標記目前正在編輯的那一天是休假日；「歷史紀錄」月曆上，被標記為休假的日期要有清楚可辨識的視覺標示
- **明確不做**（超出這次範圍，不要順手做）：
  - 不做「休假日自動排除工時/收入統計」這種邏輯運算，純視覺標記，不影響任何金額計算
  - 不碰帳戶、對帳、CSV 匯入相關程式碼
  - 不做 Decision 007 第 2 項「月曆整合收入資料顯示」，那是另一個獨立待辦，不要一起做

## 5. 資料欄位與計算規則

- 在每日 entry 物件新增一個欄位：`dayOff`（布林值，`true` / `false`）
- 沒有任何計算規則——這是純標記欄位，不參與 `cash568`/`online568`/工時/對帳/帳戶餘額的任何加總或判斷
- 沒有標記過的既有歷史資料，讀出來時 `entry.dayOff` 會是 `undefined`，程式邏輯要把它當 `false` 處理（用 `!!entry.dayOff` 或等效寫法），不要因為欄位不存在就出錯

## 6. 頁面流程與 UI 規格

### 6.1 今日輸入頁（`tab-today`）新增休假切換按鈕

- 位置：插在 `editBanner`（約 266-269 行）和 `.stats` 統計列（約 271-275 行）之間，是一個**永遠顯示**的按鈕（不受「是否為今天」影響——因為 Wei 也需要標記非今天的日期）
- HTML（新增一個 button，id 用 `dayOffToggle`）：
  ```html
  <div class="dayoff-row">
    <button type="button" class="btn small ghost dayoff-btn" id="dayOffToggle">🏖 標記為休假日</button>
  </div>
  ```
- CSS：
  ```css
  .dayoff-row { margin-bottom: 14px; }
  .dayoff-btn { width: auto; }
  .dayoff-btn.active { background: var(--pending); color: var(--accent-ink); border-color: var(--pending); }
  ```
- 行為：
  - 預設（該天未標記休假）：按鈕文字「🏖 標記為休假日」，套用既有的 `.btn.small.ghost` 樣式（outline，不填色）
  - 啟用中（該天已標記休假）：按鈕文字「☀️ 取消休假標記」，加上 `.active` class（琥珀色填滿，用 `--pending` 變數，跟打卡按鈕啟用中用的 `--pending` 是同一個顏色語意）
  - 點擊：切換該天的休假狀態 → 立刻存檔（沿用 `persistCurrentEntry()`）→ 觸發存檔回饋（沿用 `showSavedFeedback()`）→ 震動 40ms（沿用打卡按鈕的 `navigator.vibrate(40)` 寫法）→ 重新渲染月曆（`renderCalendar()`，讓月曆上的標示同步更新）
  - 這個狀態跟到「目前正在編輯的日期」（`editingDate`），不是只跟今天綁定：透過月曆點到別天、或用「回到今天」切回去，按鈕要正確顯示**那一天**的休假狀態，不能殘留上一個畫面的狀態

### 6.2 JS 邏輯（對應新增/修改的函式）

- 新增一個模組層變數：`var editingDayOff = false;`
- 新增函式 `updateDayOffButtonState()`：根據 `editingDayOff` 設定 `#dayOffToggle` 的文字內容與 `.active` class（寫法比照既有的 `updatePunchButtonState()`，約 796 行）
- 修改 `loadDateIntoForm(date)`（約 860 行）：讀出該天 entry 後，設定 `editingDayOff = entry ? !!entry.dayOff : false;`，並呼叫 `updateDayOffButtonState()`（跟現有呼叫 `updatePunchButtonState()` 放在一起）
- 修改 `persistCurrentEntry()`（約 1033 行）：在組 `entry` 物件時加一行 `entry.dayOff = editingDayOff;`
- 新增 `#dayOffToggle` 的 click handler（照 6.1 行為描述實作，可以放在 `punchBtn` click handler 附近）

### 6.3 月曆視覺標示（`renderCalendar()`，約 889 行）

- 在函式內組一個 `dayOffSet`（跟現有 `recordSet` 的寫法一樣）：
  ```js
  var dayOffSet = {};
  days.forEach(function (d) { if (d.dayOff) dayOffSet[d.date] = true; });
  ```
- 產生每個日期格子時，如果 `dayOffSet[dateStr]` 為真，`cls` 加上 `' day-off'`（跟現有 `has-record`/`today`/`selected` 的疊加寫法一致）
- CSS（新增在既有 `.cal-cell.*` 規則群組，約 141-148 行之後）：
  ```css
  .cal-cell.day-off { background: rgba(169,122,38,.22); }
  .cal-cell.selected.day-off { background: var(--accent); }
  ```
  - 第二條規則是必要的：目的是「目前選取中的日期」樣式優先權要蓋過「休假日」的底色，維持跟現有 `.selected` 的視覺優先權一致（不然選到休假日那天，格子會顯示琥珀色而不是選取中該有的主色）
  - `.has-record::after` 的小圓點跟 `.day-off` 的底色不衝突，兩者可以同時出現在同一格
- 月曆下方的圖例文字（約 314 行 `.cal-legend`）追加一段休假圖例，讓使用者看得懂顏色代表什麼：
  ```html
  <div class="cal-legend"><span class="dot"></span> 有紀錄的日期 · 點格子跳去那天編輯</div>
  <div class="cal-legend"><span class="dot" style="background:var(--pending);"></span> 休假日</div>
  ```

### 6.4 service-worker.js 版號（容易漏掉，務必做）

- 這個 App 是加到手機主畫面的 PWA，`service-worker.js` 裡的 `CACHE_NAME` 是手動版號，每次改 `index.html` 都要同步 +1，不然使用者手機上可能還在吃舊快取版本，看不到新功能（看 git log 每個功能 commit 都有同步做這件事）
- 目前是 `'taxi-ledger-v23'`，這次改完請改成 `'taxi-ledger-v24'`

## 7. 已確認決策及原因（避免代替 Wei 重新決定一次）

- **用滿版底色而不是小圓點**：呼應 Wei 在 Notion 的既有使用習慣——「休假」欄位勾選後會自動整格套色，這次刻意做成看得出「一整格變色」的視覺，比小圓點更符合 Wei 原本的直覺
- **顏色用既有的 `--pending`（琥珀色），不新增顏色變數**：專案目前用這個顏色代表「需要注意/待處理」的語意，休假日也是「跟平常不同、要特別看一眼」的狀態，語意上通，而且亮色/暗色模式的對應值已經在 `:root` 定義好了，不用再新增變數
- **點一下立即存檔，不用另外按儲存鍵**：打卡按鈕已經建立這個前例（點擊當下自動寫入 localStorage），休假標記延用同一套互動習慣，維持一致
- **休假狀態放在既有的 day entry 物件上，不開新的 localStorage key**：休假本來就是「某一天」的屬性，跟 `startTime`/收入欄位是同一層級的資料，沒必要拆成獨立資料表，而且這樣備份匯出/匯入完全不用改程式碼

## 8. 下一個開發任務（不在這次範圍內，僅供參考）

- 這次做完，Decision 007 清單裡「明確要做」的 6 項就全部完成（其餘 5 項已經在之前的版本做完，只是 `DECISIONS.md` 沒同步更新，這次順便一起訂正）
- 下一個可能的待辦是 Decision 007 第 2 項：月曆整合收入資料顯示（目前月曆格子只有「有紀錄」的點，沒顯示金額），但這次不用做，等 Wei 之後另外排

## 9. 驗收條件

- [ ] 今日輸入頁看得到「🏖 標記為休假日」按鈕，預設是未啟用樣式（outline，不填色）
- [ ] 點擊按鈕：文字變成「☀️ 取消休假標記」、背景變琥珀色、有存檔回饋（跟打卡按鈕的視覺回饋一致）、手機上有感覺到震動
- [ ] 再點一次：狀態切回未啟用，文字變回「🏖 標記為休假日」
- [ ] 切到「歷史紀錄」月曆，本月被標記休假的那天格子有清楚可辨識的琥珀色底，跟「有紀錄」的小圓點可以同時存在同一格不衝突
- [ ] 點月曆上「今天正在編輯的那天」（`.selected`）如果也是休假日，樣式要維持選取中該有的主色，不會被琥珀色蓋掉
- [ ] 點月曆上任一天跳去編輯，休假按鈕狀態要正確對應**那一天**（不會殘留切換前那天的狀態）
- [ ] 標記某一天休假，回今天看今天的休假狀態、金額欄位、總收入完全不受影響（兩天資料互相獨立）
- [ ] 重新整理頁面（或關掉瀏覽器分頁重開），休假標記還在（確認真的寫進 `localStorage`）
- [ ] 用「匯出備份」匯出一份，到另一個瀏覽器分頁「匯入備份」，休假標記要正確帶過去
- [ ] `service-worker.js` 的 `CACHE_NAME` 已經改成 `taxi-ledger-v24`
- [ ] 沒有動到帳戶、對帳、CSV 匯入相關的程式碼（可以用 `git diff` 確認改動範圍只在本規格描述的部分內）

## 10. 目前哪些檔案正在被誰改（避免衝突）

- 這次任務期間，只有 Codex 會改 `index.html` 和 `service-worker.js`，Claude／Wei 不會同時改這兩個檔案
- 完成後請自行 commit（訊息建議照現有 git log 慣例：一句話描述改了什麼 + 版號，例如「新增休假日標記(今日輸入按鈕+月曆視覺)(v24)」），**不要 push**，push 前要先讓 Wei 確認
- 完成後回報：驗收條件逐項有沒有過、實作過程中規格裡有沒有講不清楚或漏掉的地方（這是這次試驗真正想知道的事）
