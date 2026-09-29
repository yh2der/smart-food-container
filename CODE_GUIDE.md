# Smart Food Container — 程式碼說明

這份文件說明專案裡每個檔案、每一段程式在做什麼。行號對應目前的版本，之後改程式碼後行號可能會跑掉。

---

## 1. 整體架構

```
index.html            網頁外殼，只有一個 <div id="app">
src/main.js           把 App.svelte 掛到 #app 上
src/app.css           全站共用的樣式（字型、顏色變數）
src/App.svelte        主頁面：所有狀態、計算邏輯、右邊的 Testing 面板、Info 視窗
src/lib/FoodStatus.svelte   左邊的 Device UI：容器圖、放大的蓋子螢幕、蓋子按鈕
vite.config.js        Vite 設定（base: './' 讓 GitHub Pages 子路徑能用）
.github/workflows/deploy.yml   推到 main 時自動 build 並部署到 GitHub Pages
```

資料怎麼流動：

```
App.svelte（擁有所有資料）
   │  用 props 把資料傳下去：食物名稱、天數、狀態、燈色、細菌量...
   ▼
FoodStatus.svelte（只負責顯示）
   │  使用者按蓋子上的按鈕時，呼叫 App 傳進來的函式（onAdjust、onClear）
   ▼
App.svelte 修改資料 → Svelte 自動重新計算 → 畫面自動更新
```

**重點觀念：** 所有「真正的資料」都放在 `App.svelte`。`FoodStatus.svelte` 只拿資料來畫圖，自己不修改。唯一的例外是「螢幕要顯示狀態還是歷史圖」，這只跟螢幕本身有關，所以放在 FoodStatus 裡。

---

## 2. 用到的 Svelte 5 語法

| 語法 | 意思 |
|---|---|
| `$state(初始值)` | 會變動的資料。值一改，用到它的畫面就自動更新 |
| `$derived(算式)` | 由其他資料**算出來**的值。來源改變時自動重算，不用手動更新 |
| `$props()` | 元件從父元件接收的資料 |
| `{#if ...} {:else if ...} {:else} {/if}` | 條件顯示 |
| `{#each 陣列 as 項目}` | 迴圈，對陣列裡每一項產生一段畫面 |
| `onclick={函式}` | 點擊時執行函式 |
| `class:名稱={條件}` | 條件成立時加上這個 CSS class |
| `{變數}` | 把變數的值顯示在畫面上 |
| `<style>` | 只作用在這個元件內的 CSS，不會影響其他元件 |

---

## 3. `src/App.svelte` — 主頁面

### 3.1 食物情境資料（第 4–27 行）

`SAMPLES` 定義 4 個食物情境，是整個模擬的「劇本」：

| 欄位 | 意思 |
|---|---|
| `name` | 螢幕上顯示的名稱 |
| `label` | Testing 面板按鈕上的名稱 |
| `icon` | 按鈕上的 emoji |
| `days` | 拍照辨識後**預估**還能放幾天 |
| `bacteria` | 放進去時細菌感測器的初始讀數（% ，100% = 不安全） |
| `growth` | 每過一天細菌量增加多少 % |
| `note` | 按鈕上的小字說明 |

四種食物用不同的數字，模擬出不同的結果：

| 食物 | days | 細菌變化 | 結果 |
|---|---|---|---|
| White Rice | 3 | 10 → 40 → 70 → 100 | 照預估正常變壞 |
| Fried Rice | 1 | 40 → 100 | 隔天就壞 |
| Chicken Soup | 4 | 15 → 60 → 100 | 預估 4 天，但第 2 天感測器就判定壞了 |
| Salad | 2 | 5 → 20 → 35 | 細菌一直不高，但天數到了還是過期 |

### 3.2 狀態變數（第 29–37 行）

| 變數 | 意思 |
|---|---|
| `selectedFood` | Testing 面板目前**選到**哪個食物（還沒拍照） |
| `detectedFood` | 容器裡**目前的**食物；`null` 代表空的 |
| `daysLeft` | 預估剩幾天（還沒考慮感測器） |
| `isAnalyzing` | 是否正在「Analyzing food...」 |
| `showInfo` | Info 視窗是否打開 |
| `bacteriaHistory` | 每天的細菌讀數陣列，最後一筆是今天，例如 `[10, 40, 70]` |
| `simDay` | 放進容器後過了幾天，顯示在 Testing 面板 |
| `isPlaying` | 自動播放是否進行中 |
| `timer` | `setInterval` 的編號，停止播放時要用。不需要顯示在畫面上，所以不用 `$state` |

`selectedFood` 和 `detectedFood` 分開，是因為「在面板上選」和「容器真的辨識到」是兩件事，要按 Take Photo 才會從前者變成後者。

### 3.3 算出來的值（第 39–66 行）

這些都用 `$derived`，所以只要源頭資料改變就自動更新。

- **`foodName`（第 39 行）**：從 `detectedFood` 查出食物名稱，沒有食物就是 `null`。
- **`bacteria`（第 41 行）**：今天的細菌讀數，也就是 `bacteriaHistory` 最後一筆。陣列是空的就當作 0。
- **`bacteriaLevel`（第 42 行）**：把讀數分成三級：`100 以上 → 'high'`、`50 以上 → 'rising'`、其他是 `'normal'`。
- **`shownDays`（第 45–47 行）**：螢幕上**實際顯示**的天數。這是整個設計的核心規則：**感測器只能讓情況變差**。
  - High → 直接 0 天
  - Rising → 最多 1 天，`Math.min(daysLeft, 1)`
  - Normal → 照預估的 `daysLeft`
- **`status`（第 49–61 行）**：螢幕上的狀態文字，從上往下檢查，先符合的先用：
  1. 沒有食物 → 空字串
  2. 細菌 High → `SPOILED`
  3. 天數 0 → `EXPIRED`
  4. 細菌 Rising → `EAT TODAY`
  5. 3 天以上 → `SAFE`
  6. 其他（1–2 天）→ `EAT SOON`
- **`ledColor`（第 64–66 行）**：LED 燈環顏色。沒食物是灰色，3 天以上綠色，2 天橘色，1 天以下紅色。用的是 `shownDays`，所以感測器的判斷也會反映在燈上。

### 3.4 函式（第 68–121 行）

- **`takePhoto()`（第 68–78 行）**：模擬拍照辨識。
  1. 先停止自動播放。
  2. `isAnalyzing = true`，面板顯示「Analyzing food...」，按鈕暫時不能按。
  3. `setTimeout` 等 1.2 秒，假裝 AI 在辨識。
  4. 把選到的食物放進容器，設定預估天數和細菌初始值，天數計數歸零。

- **`advanceDay()`（第 80–88 行）**：過一天。
  - 食物還沒壞（`shownDays > 0`）時：預估天數減 1（最低 0），`simDay` 加 1，並新增一筆細菌讀數（今天的值 + 這種食物的 `growth`，最高 100）。
  - 如果過完這天食物已經是 0 天，就停止自動播放。

- **`setBacteria(value)`（第 91–93 行）**：Testing 面板的 Normal / Rising / High 按鈕用的，直接把**今天**的讀數改成指定值，方便 demo。

- **`togglePlay()`（第 96–103 行）**：Play / Pause。播放時用 `setInterval` 每 1000 毫秒（1 秒）呼叫一次 `advanceDay`。

- **`stopPlay()`（第 105–108 行）**：停止計時器，`isPlaying` 設回 false。

- **`adjustDays(delta)`（第 111–113 行）**：蓋子上的 − / + 按鈕。`delta` 是 -1 或 +1，結果限制在 0–14 天之間。注意它改的是 `daysLeft`（預估），所以感測器是 Rising 或 High 時，按 + 畫面也不會超過感測器允許的天數。

- **`clearFood()`（第 115–121 行）**：蓋子上的 Clear 按鈕。清空容器，所有資料回到初始狀態。

### 3.5 畫面：左邊 Device UI（第 125–138 行）

放一個 `<FoodStatus>` 元件，把需要顯示的資料都傳進去：

- `daysLeft={shownDays}`：傳的是**已經考慮感測器**的天數，不是原始預估。
- `foodType={detectedFood}`：用來決定容器裡畫哪種食物。
- `onAdjust={adjustDays}`、`onClear={clearFood}`：把函式傳下去，讓蓋子按鈕可以通知 App 修改資料。

### 3.6 畫面：右邊 Testing 面板（第 140–201 行）

由上到下：

1. **標題和 Info 按鈕**（第 141–145 行）：專案名稱。點「i」會把 `showInfo` 設成 true，打開說明視窗。
2. **食物情境選擇**（第 147–163 行）：用 `{#each}` 把 `SAMPLES` 的 4 種食物各做成一個按鈕。選到的那個會加上 `selected` class，顯示外框。
3. **Take Photo**（第 165–168 行）：分析中時按鈕停用，並顯示「Analyzing food...」。
4. **時間模擬**（第 172–178 行）：顯示目前第幾天，以及 Advance 1 Day 和 Play/Pause 按鈕。沒食物、食物已經 0 天、或正在分析時，按鈕停用。
5. **細菌感測器測試按鈕**（第 182–193 行）：顯示目前讀數。三個按鈕分別把讀數設成 20 / 60 / 100，目前等級的按鈕會有外框。
6. **作者和 write-up 連結**（第 197–200 行）。

### 3.7 Info 視窗（第 204–232 行）

`showInfo` 為 true 時才顯示。

- 外層 `.backdrop` 是半透明黑色背景，蓋住整個畫面。
- 點背景會關閉視窗。`e.target === e.currentTarget` 是在確認點的是背景本身，而不是視窗裡的內容。
- 視窗內容說明所有模擬操作和蓋子按鈕的功能。

### 3.8 樣式（第 234–351 行）

- **`main`**：左右兩欄的 flex 排版。
- **`.device-ui`**：左邊佔 78% 寬度，內容置中。
- **`.testing-ui`**：右邊佔剩下的寬度，有左邊框，內容垂直排列。
- 其他是按鈕、文字大小、分隔線、Info 視窗的外觀。

---

## 4. `src/lib/FoodStatus.svelte` — 左邊的 Device UI

分成兩塊：左邊是**實體容器示意圖**（表示 UI 在容器的哪個位置），右邊是**放大的蓋子螢幕和按鈕**（實際操作的介面）。

### 4.1 接收資料和內部狀態（第 1–25 行）

- **第 2–13 行**：用 `$props()` 接收 App 傳來的所有資料和函式。
- **第 15 行 `dayLabel`**：1 天顯示 `DAY LEFT`，其他顯示 `DAYS LEFT`（單複數）。
- **第 18 行 `showHistory`**：螢幕目前顯示「狀態」還是「細菌歷史圖」。這是 FoodStatus 自己的狀態，因為只影響螢幕顯示。
- **第 20 行 `LEVEL_COLORS`**：三個細菌等級對應的顏色，畫長條圖用。
- **第 21 行 `levelOf(v)`**：判斷一個讀數屬於哪個等級（和 App 裡的規則相同），用來決定每根長條的顏色。
- **第 24 行 `barWidth`**：長條圖每根的寬度。天數越多，每根越窄，最寬 30。

### 4.2 容器示意圖（第 29–76 行）

- **蓋子（第 32–37 行）**：灰色長方形。裡面的黑色小方塊代表螢幕位置，旁邊 4 個小長條代表 4 顆按鈕。只是示意，不能按。
- **容器本體（第 41–74 行）**：
  - `class:blink={bacteriaLevel === 'high'}`：細菌 High 時加上 `blink` class，讓燈環閃爍。
  - `style="--led: {ledColor}"`：把燈色設成 CSS 變數 `--led`，下面的 CSS 用這個變數上色。
  - **`.ring`（第 42 行）**：LED 燈環，一條會發光的色帶。
  - **`.sensor`（第 43–44 行）**：蓋子內側的細菌感測器，以及文字標籤。
  - **食物圖（第 45–73 行）**：依 `foodType` 用 `{#if}` 決定畫哪種食物。每種都是簡單的 SVG：一個橢圓代表食物本體，幾個小圓或方塊代表配料。
- **說明文字（第 38、75 行）**：「↑ Lid screen + buttons」和「↑ 360° LED ring」，對應作業要求的「標示 UI 在實體物品的哪裡」。

### 4.3 放大的蓋子螢幕（第 79–117 行）

螢幕有三種顯示模式，用 `{#if}` 切換：

1. **歷史圖模式（第 82–105 行）**：有食物而且 `showHistory` 為 true 時。
   - 用 SVG 畫圖，座標範圍 200 × 120。
   - **第 86–89 行**：兩條虛線門檻。y=10 是 100%（紅），y=60 是 50%（橘）。
   - **第 90–101 行**：每天一根長條。SVG 的 y 軸是往下增加的，所以長條頂端是 `110 - value`、高度是 `value`，這樣長條就會從底部往上長。下方標示 D0、D1…
   - **第 103–105 行**：顯示存放了幾天和目前讀數。
2. **狀態模式（第 106–113 行）**：有食物時的預設畫面。
   - 食物名稱（轉大寫）
   - 大大的天數，`Math.max(daysLeft, 0)` 確保不會顯示負數
   - DAY(S) LEFT
   - 狀態文字，顏色和燈環相同
   - 細菌讀數；不是 Normal 時加上 `alert` class，變紅色粗體
3. **空的（第 114–115 行）**：沒有食物時顯示 `NO FOOD DETECTED`。

### 4.4 蓋子按鈕（第 119–127 行）

| 按鈕 | 做什麼 | 什麼時候停用 |
|---|---|---|
| − | 呼叫 `onAdjust(-1)`，也就是 App 的 `adjustDays(-1)` | 沒食物，或正在看歷史圖 |
| + | 呼叫 `onAdjust(1)` | 同上 |
| History / Status | 切換 `showHistory`，按鈕文字也跟著變 | 沒食物 |
| Clear | 呼叫 `onClear`，也就是 App 的 `clearFood()` | 沒食物 |

看歷史圖時停用 − / +，是因為改天數的結果在歷史圖上看不到，避免使用者按了卻以為沒反應。

### 4.5 樣式（第 131–332 行）

幾個重點：

- **`.lid` 的 `transform: perspective(600px) rotateX(35deg)`（第 172 行）**：讓蓋子往後傾斜，看起來像從斜上方看容器。這就是「假 3D」效果，不需要 Three.js。
- **`.box`（第 195–207 行）**：半透明淺藍色的盒子，模擬透明容器。
- **`.ring`（第 210–221 行）**：
  - `background: var(--led)` 用燈色填滿色帶。
  - `box-shadow: 0 0 16px 4px var(--led)` 產生同色的光暈，看起來像在發光。
  - `transition` 讓顏色變化時有 0.4 秒的淡入淡出。
- **`.blink .ring` 和 `@keyframes blink`（第 224–232 行）**：細菌 High 時燈環每 0.8 秒閃一次（透明度在 1 和 0.2 之間變化）。
- **`.screen`（第 261–270 行）**：黑底白字、等寬字體，模擬電子螢幕。
- **`.days`（第 277–282 行）**：72px 粗體，是整個畫面最顯眼的元素。

---

## 5. 其他檔案

- **`index.html`**：網頁本體，只有一個空的 `<div id="app">`，所有內容都由 Svelte 產生。`<title>` 還是 `smart-food-container`，可以改成「Smart Food Container」，瀏覽器分頁的名稱會比較好看。
- **`src/main.js`**：程式進入點。載入 `app.css`，再用 `mount()` 把 `App.svelte` 放進 `#app`。
- **`src/app.css`**：Vite 範本附帶的全站樣式，定義字型和顏色變數（例如 `--border`、`--accent`、`--text-h`），元件裡會用到這些變數。裡面還有一些範本留下、目前沒用到的樣式（`.hero`、`#center`、`#next-steps` 等），不影響功能。
- **`src/lib/Counter.svelte`**：Vite 範本附帶的計數器元件，專案裡沒有用到，可以刪掉。
- **`vite.config.js`**：
  - `plugins: [svelte()]` 讓 Vite 看得懂 `.svelte` 檔。
  - `base: './'` 讓 build 出來的檔案用相對路徑。這樣部署到 `帳號.github.io/repo名稱/` 這種子路徑時，JS 和 CSS 才載入得到。
- **`.github/workflows/deploy.yml`**：GitHub Actions 自動部署。每次推到 `main`，GitHub 就會：
  1. 下載程式碼
  2. 安裝 Node 22
  3. `npm ci` 安裝套件
  4. `npm run build` 產生 `dist/`
  5. 把 `dist/` 上傳並部署到 GitHub Pages

---

## 6. 一個完整操作的流程範例

以「選 Chicken Soup → Take Photo → Play」為例：

1. 點 Chicken Soup：`selectedFood = 'chicken-soup'`，按鈕出現外框。
2. 點 Take Photo：`isAnalyzing = true`，面板顯示 Analyzing food...
3. 1.2 秒後：
   - `detectedFood = 'chicken-soup'`，`daysLeft = 4`，`bacteriaHistory = [15]`
   - 自動算出：`bacteria = 15`，等級 normal，`shownDays = 4`，`SAFE`，綠燈
   - 螢幕顯示 CHICKEN SOUP / 4 / DAYS LEFT / SAFE / BACTERIA: 15% NORMAL
4. 點 Play：每秒呼叫一次 `advanceDay()`。
   - **第 1 天**：`daysLeft = 3`，細菌 15+45 = 60 → Rising → `shownDays = min(3, 1) = 1` → `EAT TODAY`，紅燈
   - **第 2 天**：`daysLeft = 2`，細菌 60+45 = 105，上限 100 → High → `shownDays = 0` → `SPOILED`，紅燈閃爍，自動停止播放
5. 點蓋子上的 History：螢幕變成三根長條（15 綠、60 橘、100 紅）。
6. 點 Clear：全部歸零，螢幕顯示 NO FOOD DETECTED，燈變灰。

這個例子剛好說明了設計重點：預估還有 2 天，但感測器發現已經壞了，所以以感測器為準。
