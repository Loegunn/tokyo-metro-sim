# 東京地下鐵車內顯示屏 · 交接文件（2026-09-09）

呢個係 Loegunn 嘅興趣project：單一 HTML 檔案嘅車廂資訊顯示屏模擬器（次頁「つぎは」LCD風格＋語音報站＋橫向路線strip），原本淨係山手線，而家改成可以喺同一個網頁切換多條線。長遠目標係整成一個app俾「鐵膠」（鐵道迷）玩，仲未有明確時間表，用戶自評依家「10分滿分得6分，好多細節未執」。

## 版本歷程
1. `無bgm嘅yamanotev5.html`（用戶原本上載）— 純山手線，環狀，冇地圖。
2. v6 試過加「真實形狀路線圖」。**用戶反應：「有啲醜」，已經放棄呢個方向，唔好再撿返嚟做**。
3. 現行版本 `東京地下鐵車內顯示屏.html`：由v5開始，做咗「多路線可切換」嘅重構，加咗銀座線、日比谷線、東西線、有楽町線，加咗開門方向色彩區分同乘客／全螢幕模式呢兩個UX功能，修正咗兩輪報站相關嘅真bug。而家有五條線：山手線／銀座線／日比谷線／東西線／有楽町線。前四條線嘅開門方向資料都已經有咗；有楽町線暫時未有，全部標null。

## 部署方式：用戶自己用GitHub Pages host＋GitHub Desktop
- 用戶試過將HTML檔案喺iPhone用Files app／Share Sheet打開，發現淨係觸發到iOS嘅Quick Look預覽，唔係真正嘅Safari分頁，好多功能未必work得到。
- 用戶**明確表示唔想用Claude託管嘅Artifact網站**（「我唔想用你託管網站」），改用自己個GitHub帳號開GitHub Pages（免費static hosting，Public repo，Settings→Pages→Deploy from branch→main→/root）。
- **重要（俾之後接手嘅Claude睇）：以後交付呢個project嘅更新，預設做法係俾用戶一個完整嘅`index.html`檔案（用SendUserFile），唔好推去Claude Artifact嗰條link做主要分發方式。**
- 用戶之後裝咗**GitHub Desktop**，clone咗個repo（叫`tokyo-metro-sim`）落自己部電腦，之後更新流程係：喺Finder覆蓋本地資料夾入面嘅`index.html` → 開返GitHub Desktop → 「Commit to main」→「Push origin」，GitHub Pages大約一分鐘後自動重新部署。
- 用戶已經用自己部iPhone經GitHub Pages網址測試過，回報「完全沒問題」——第一次有真機（唔係Playwright模擬）確認咗`primeSpeech()`嘅iOS解鎖修正、以及第一輪嘅兩個報站bug修正全部work。第二輪修正（終點掉頭方向、暫停漏聲，見下面）仲未有真機report。
- **IP考量提一提**：Public repo代表個code而家任何人都search到、睇到。用戶未有再問，暫時淨係記低。

## 現行架構（做嘢之前建議睇實際檔案，呢度淨係摘要）
- `LINES` 呢個object入面每條線一個entry：`id/name/badge/topology('loop'|'linear')/color/colorDeep/dirLabels/stations/footnote`。加多條線基本上只係加data，唔使改架構——有楽町線就係咁加返嚟嘅，冇改過`order()`/`at()`/`switchLine()`半隻字。
- `order()`／`at()` 兩個function做咗拓撲通用化：`loop`（山手線）wrap一個環；`linear`（銀座線、日比谷線、東西線、有楽町線）用「虛擬長度 2×(N-1)」嘅來回序列模擬去到總站自動调头，唔使額外狀態機。**注意**：`linear`呢個topology假設條線淨係得兩個總站、冇支線；如果之後想加有支線嘅線（例如丸ノ内線嘅方南町支線、千代田線嘅北綾瀬支線、半蔵門線／副都心線本身雖然無支線但涉及同其他線嘅track-sharing直通運轉，都要留意呢個topology夠唔夠用），要另外諗過個資料結構先做得。
- **`effOuter(i)`（2026-09-09加，修Bug 3用）**：`linear`路線嘅`order()`會用「虛擬長度2×(N-1)」嘅來回序列模擬自動掉頭，但個站序入面邊段係「去程」邊段係「返程」淨係喺`order()`內部知道，全局變數`outer`本身唔會跟住掉頭而更新。`effOuter(i)`計返喺idx=i嗰陣，實際係咪仲同`outer`嗰個初始方向一致（`(m<=N-1)===outer`）。`doorSideText()`、`enter()`嘅報站side、`setDirLabel()`統一用`effOuter(idx+1)`，唔可以再直接用`outer`。`loop`拓撲（山手線）冇呢種來回結構，`effOuter`會直接原封不動返返`outer`。
- 切換路線用 `switchLine(id)`，會reset晒 idx/t/phase/outer/dirMode，換色、換徽章文字、換footnote、rebuild方向掣同起點dropdown。下拉揀線嘅list（`selLine`）由`Object.values(LINES)`自動生成，加新線唔使手動改呢部分。
- 開門方向資料格式：`door:[A,B]`——loop個case係`[内回り側,外回り側]`；linear個case係`[較細編號方向嗰邊側,較大編號方向嗰邊側]`。唔肯定嘅站一律 `null`；有兩種特殊軟性標記：`'R?'`（銀座線浅草站）、`'LR'`（真係兩邊都有可能，例如日比谷線北千住、東西線中野／妙典／西船橋）。`doorSideText(st,i)`會處理晒呢四種case，同時回傳個純`side`值俾`paintDoorHighlight()`用。
- 每個報站scheduling有個 `genId` 代數標記（切換路線／跳站／暫停／靜音／收埋分頁都會令`genId`增加——已經摺咗入`stopSpeech()`本身，唔使逐個call site記得加），`announce()`同close-phase嘅setTimeout call speak()之前會check代數有冇變，變咗就唔再開聲。

## 兩輪報站相關嘅真bug修正

### 第一輪（用戶用screenshot實測搵出嚟；已喺真iPhone確認work）
**Bug 1**：`enter(p)`入面`idx++`同計算報站用嘅`n=at(idx+1)`次序調轉咗，搞到啱啱departure嗰句報站用咗舊idx。**修正**：`idx++`搬去計`n`之前。
**Bug 2**：`speak()`每次都即刻cancel，正常速度下轉乘名單長過8秒會俾打斷。**修正**：`speak()`喺速度×1時排隊、×3/×8保留即刻cancel；新加`stopSpeech()`統一處理真正要即刻收聲嘅場合。

### 第二輪（用戶貼咗ChatGPT寫嘅程式評閱，逐樣驗證後先落手改；已用Playwright驗證，未有真機report）
**Bug 3**：直線來回路線（銀座線／日比谷線／東西線）行到終點自動掉頭之後，方向標籤同開門邊冇跟住變。**根源**：`order()`會自動幫車掉頭，但`outer`呢個全局變數只代表用戶最初揀嘅方向，掉頭後唔會更新，而所有睇方向嘅地方都直接用緊呢個冇更新嘅`outer`。**修正**：加咗`effOuter(i)`（見上面「現行架構」section）。**測試**：東西線神楽坂喺去程（idx=3→focus4）同返程（idx=39→focus40）分別測過，方向標籤同開門邊都跟返啱。
**Bug 4**：暫停／靜音／收埋分頁之後，啱啱排緊隊嘅delayed報站可能照樣播出嚟。**根源**：`stopSpeech()`冇更新`genId`，已經排咗程嘅`setTimeout`會照樣觸發。**修正**：`genId++`摺咗入`stopSpeech()`本身。
**未做**：ChatGPT提到嘅兩個細位（英文轉乘名單封頂3個vs日文冇上限；停站期間大字即刻跳去再下一站）驗證過都係真，但影響細，未處理。

## 開門方向色彩區分／乘客全螢幕模式
- `paintDoorHighlight(side)`：`side==='L'`或`'LR'`highlight左門；`'R'`／`'R?'`／`'LR'`highlight右門；淨係「肯定」case（`L`／`R`）先加`flash`。`prefers-reduced-motion`有fallback。
- `#btnPassenger`：toggle `.unit`嘅`passenger` class，隱藏`.ctl`／`.audio`，嘗試`requestFullscreen()`，撳多次或Esc都自動同步。

## 各線資料摘要
- **銀座線**（G01渋谷～G19浅草，19站）：官方編號一度寫反咗方向，已經用用戶資料改正。開門方向全部19站已填。
- **日比谷線**（H01中目黒～H22北千住，22站）：開門方向全部22站已填；北千住往中目黒方向`'LR'`。
- **東西線**（T01中野～T23西船橋，23站）：開門方向全部23站已填；中野／妙典／西船橋`'LR'`，神楽坂兩方向唔同邊。
- **有楽町線**（Y01和光市～Y24新木場，24站，2026-09-09新加）：官方編號同路線總覽Fetch官方頁面核實；轉乘資料逐個複雜站（和光市、池袋、飯田橋、市ケ谷、東池袋、永田町、月島）個別官方頁面交叉核對——證實和光市冇JR轉乘、東池袋官方冇認都營大江戸線嗰個轉乘（跟返保守原則冇加）；永田町↔赤坂見附係官方認嘅「唔同站名轉乘」（同銀座線、日比谷線嗰種一樣）。JR具體線名（池袋/飯田橋/市ケ谷/有楽町/新木場）官方頁面淨係寫「JR線」冇講邊條，係跟其他線自己嘅車站資料交叉比對出嚟。**開門方向：用戶未提供，全部24站`null`（未確認），等用戶下次提供資料。**

## 測試方法學嘅教訓
- 驗證「每個站都有報站」要用`page.add_init_script`將`window.requestAnimationFrame`覆寫做冇效，唔可以同手動call嘅`window.loop()`並存（會令`dt`變負數觸發NaN錯誤，呢個係測試方法問題，唔係app bug）。
- 驗證報站文字／排隊行為要**保留返真實rAF**，直接monkey-patch `speechSynthesis.speak`/`cancel`。
- 驗證門開邊／方向標籤呢類「跟住idx變」嘅顯示邏輯，要留意`paint()`裡面嘅`focus`永遠都係`at(idx+1)`，唔係`at(idx)`——第一次測試Bug 3嗰陣中過呢個伏。
- **外部（第三方AI）code review嘅處理原則**：唔照單全收，逐項用Playwright實測驗證先落手改；證實嘅先改，未證實／純屬設計意見嘅記低但唔急住做。

## 報站聲音嘅測試同已知限制
- 五條線全部驗證過：每個站到站都會觸發到 `speak()`/`announce()`，冇漏一個，0個JS error。
- iOS Safari嘅`primeSpeech()`修正**已經喺真iPhone確認work**。Android Chrome要裝日文TTS語音包，未有真機report。
- 第二輪修正（Bug 3、Bug 4）同有楽町線新加嘅內容，暫時**未有真機report**，淨係Playwright驗證過。

## 外部review意見嘅評估
- 第一份（表格形式）：真係值得做嘅淨係「開門方向色彩區分」同「乘客全螢幕模式」，已完成。
- 第二份（ChatGPT code review）：真係confirmed嘅有Bug 3、Bug 4，已修正；兩個細位證實真但影響細，未處理；其餘多數係已經做咗嘅嘢或者主觀設計意見。

## IP／侵權考量（用戶問過，未有結論，暫時擱置）
Public repo代表個code而家任何人都search到、睇到，同之前「純私人」已經唔同——用戶未有再問，暫時淨係記低。

## 下一步可能方向（未拍板，睇用戶點揀）
- 有楽町線開門方向資料等用戶提供。
- 第二輪修正（Bug 3、Bug 4）未有真機report，可以等下次用戶更新之後順手試下。
- 執返ChatGPT提到、但未處理嘅兩個細位：英文轉乘名單封頂3個vs日文冇上限；停站期間大字顯示邏輯。
- 加多幾條線：冇支線嘅線（半蔵門線、南北線、副都心線）可以直接加，零架構改動；有支線嘅線（丸ノ内線、千代田線、都營大江戸線）要先諗過拓撲資料結構。
- 執靚視覺／操作細節：小螢幕控制列排版、無障礙/語言一致性，中低優先，未開始。
- 諗個「app」大致形態（依家係一頁切換線，定係想要個首頁揀線）。
- 官方編號方向每加新線都要double check，Fetch官方總覽頁＋逐個複雜站個別頁面交叉核對嘅做法（呢次加有楽町線用嘅方法）可以繼續用。
