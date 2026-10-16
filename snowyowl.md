---
name: snowyowl
description: |
  用於 snowyowl（台股／美股資料、選股、回測）的任何任務，或廣義的台美股資料／篩股／
  策略需求。涵蓋——
  選股／篩股／screen Taiwan or US stocks：積木 blocks、api.Block、con_tw／con_us、
  screener_tw／screener_us、get_table_tw／get_table_us、積木條件、選股條件、
  多條件 AND、股池 pool。
  歷史與矩陣資料：api.MSMP／api.USMSMP、日_K／週_K／月_K、adj 還原股價、漲跌幅、
  成交量／成交額、市值。
  表格查詢：api.Data／api.USData、get(table)、清單／分K／逐筆明細。
  基本面：月營收、季財報、EPS、本益比 PER、ROE、毛利率、股利。
  籌碼：三大法人、外資／投信／自營、融資融券、八大行庫、券商分點、大戶持股、籌碼集中度。
  技術面：均線、MACD、RSI、KD、布林、突破。
  盤中資料：分K、逐筆明細、美股即時快照 api.USData.get_snapshots。
  回測：api.bt、backtest、事件式策略、進出場、停損停利、部位大小、績效 KPI、
  積木 sig 餵回測。
  台指期：MSMP._fut_TXF、期貨 1 分 K。
---
> **這份 skill 需要本機已安裝 snowyowl 套件**：
>
> ```bash
> pip install git+https://github.com/SnowyOwlStrategy/SnowyOwl.git
> ```
>
> 它教 AI 怎麼寫 snowyowl 的 Python，不是自己去打 API——沒裝套件的話
> 裡面的 `so.login()` 會直接 ModuleNotFoundError。
>
> 完整文件站：<https://snowyowlstrategy.github.io/> ｜ 原始碼：<https://github.com/SnowyOwlStrategy/SnowyOwl>

# snowyowl 積木選股

台股 416 個（含 11 個以通用積木寫成的具名積木）、美股 260 個積木。**規格的唯一真相來源是 `get_table_*()`**，不是 Excel、不是 `list_conditions`。

完整文件在 <https://snowyowlstrategy.github.io/>，索引見 <https://snowyowlstrategy.github.io/llms.txt>，全文串接見 <https://snowyowlstrategy.github.io/llms-full.txt>。
本檔可從 <https://snowyowlstrategy.github.io/snowyowl.md> 取得最新版。

## 起手式

```python
import snowyowl as so
api = so.login('user@x.com', 'pwd')   # 帳密登入
# 或 so.login(type='google')          # 開瀏覽器，帳號在瀏覽器裡選（apple 同理）
B = api.Block
```

`so.login(person_id, person_pwd, type='account')`：
- **帳密**（預設 `type='account'`）：`person_id` 與 `person_pwd` 兩個都要，缺一會印
  「登錄失敗」不登入。**`so.login()` 空手呼叫會失敗**。
- **google / apple**（`type='google'` / `'apple'`）：開瀏覽器讓使用者選帳號，不必給帳密。
  首次要在瀏覽器認證，之後 72 小時內走本機快取免認證（帳密登入不快取）。
- 登入結果會快取在 `~/.snowyowl/auth.json`；`api.logout()` 會清掉，`force=True` 強制重登。

登入約 10-15 秒。積木的資料來源是 `login` 內部呼叫 `bind_api()` 注入的，
**不要繞過它**去 `from snowyowl.blocks.data_context import ...`：

- `import api` → `ImportError`（那個名字不存在）
- `import msmp` / `usmsmp` → 沒注入時**安靜回 `None`**，
  之後才在別的地方炸成 `AttributeError`，離現場很遠、很難追

一律走 `api.Block`。

## 先分流：這個問題該走哪條路

被 `/snowyowl <問題>` 呼叫時，先判斷是哪一類，**不要一律去找積木**：

| 使用者問的 | 走哪 |
|---|---|
| **單純事實查詢**——今天漲幅 5% 以上、台積電最近一個月股價、成交金額前 10 名、誰漲停 | **`api.MSMP` 一行算完**（見下方配方） |
| **條件組合選股**——突破 20 日高 **且** 成交金額佔大盤 >1% **且** 外資買超 | **積木** `api.Block`（或 MCP 的 `screen`） |
| **要一支能重複跑的腳本／回測** | 寫 Python，見 [backtest.md](https://snowyowlstrategy.github.io/backtest/) 起五頁 |

積木是「把多個條件 AND 起來篩股」用的，**不是查數據用的**。
硬拿積木回答事實問題，常常找不到剛好對的，或找到名稱像但語意不同的（見下方 `o6` 的例子）。

### 回測類請求的預設做法（使用者不會講，一律照做）

使用者只會用白話描述策略（「月營收成長最高的 20 檔」「成交量大的」「5 日線穿過 10 日線」），
不會知道積木名稱、`tick_map` 這些細節。下面這些由你主動處理，不必等他說：

1. **先找對應的配方照抄。** [recipes.md](https://snowyowlstrategy.github.io/recipes/) 有七份實跑過的完整腳本，對應 sample.md 的案例。
   找到最接近的那份直接改參數，**不要先讀 `llms-full.txt`**（845 KB，讀完要很久），
   缺什麼細節再查單篇 docs。
2. **條件先對應到積木。** 選股、過濾、進出場條件，先到 [block-reference-tw.md](https://snowyowlstrategy.github.io/block-reference-tw/)
   （美股看 [block-reference-us.md](https://snowyowlstrategy.github.io/block-reference-us/)）用關鍵字找名稱相符的積木，用 `api.Block.con_tw(...)` 組起來、
   `screener_tw` 跑出 `sig`，再照配方二餵進 `Core`（`feed_data(..., columns=['sig'])`）。
   參數只能用該積木列出的合法值。找不到對應積木的條件（例如自己合成的多因子分數）才手寫。
   **回覆時列出用了哪些積木、參數是什麼，哪些條件是手寫的。**
3. **跑完一律產儀表板。** 腳本裡用 `rp.dashboard(path='<策略名>_dashboard.html', view='web')`，
   notebook 裡用 `rp.dashboard()`。回覆時給出檔案路徑。
4. **指標要上圖就寫進 `self.char`／`self.char1`。** 均線等指標值寫進 `self.char[...]`（主圖）
   或 `self.char1[...]`（副圖），儀表板個股分析的 K 線才會畫出來。
5. **期貨（TXF）沒有積木**，進出場寫在 `on_bar`，另外三件事一定要做：
   - `on_init` 設 `self.tick_map = [(np.inf, 1)]`。預設是股票跳動（≥1000 以 5 為單位），
     限價、停損、`compel_now` 的價格都會被修到 5 點的倍數。
   - 結算棒（`row['DateTime'] == row['Settlement']`）用 `compel_now` 帶 `row['Close']` 強平，價格不能給 0。
   - 多空反手**拆成兩張單**：先 `-self.global_qty` 平倉（rule `net`），再開新倉（`bk`／`sk`）。
     合成一張的話損益相同，但舊倉的出場規則會被標成新倉的名字，進出場規則績效會錯。

### 回測類請求：先照做，跑完主動給三個數字

使用者說「幫我回測 X」時，**照他的規格做，不要自作主張改**（他說每檔 10 萬就是 10 萬，
不要擅自改成目標權重輪動）。但跑完**主動**附上這三個，不必等他問：

1. **毛損益與成本佔比**——`(pnl+cost).sum()` 與 `cost/Invest`。
   日頻策略常常毛損益是正的、虧損全來自成本。看到負報酬就說策略沒用是誤判。
2. **回測期間實際是哪一段**——日頻硬上限 1000 筆（約 4.1 年）。
   使用者以為的「近十年」實際只有四年。
3. **曝險率**——`(rp.daily['position'].abs()>0).mean()`。

然後**才**下結論。結論要先分「參數問題」還是「結構問題」：

| 症狀 | 多半是 | 該做的事 |
|---|---|---|
| 曝險率 < 80% 且輸給 buy&hold，標的又是大多頭 | **結構問題** | 直說掃參數救不了，空手期就是機會成本。要改方向（換標的、加空單、或接受較低報酬換低 MDD） |
| 毛損益為正、淨損益為負 | **成本問題** | 降頻、減少進出，不是調訊號參數 |
| 交易次數 < 30 | **樣本不足** | 結論不可信，先放寬條件或拉長期間 |
| 上述都不是，報酬只是普通 | **可能是參數問題** | 這時掃參數才有意義，且要同時輸出 `TotalTrades` 與曝險率避免過度配適 |

**不要一被要求「找更好的參數」就直接開始掃。** 先看曝險率——
實測 0050 的 MA 策略掃 95 組參數，贏過 buy&hold 的是 **0 組**，
因為所有組合平均曝險只有 72.7%。這種時候掃參數是浪費時間，該講的是結構性原因。

### 有哪些資料（先看這裡，不要當場翻文件）

**台股矩陣 `api.MSMP`，25 張**（日期 × 股票代號，括號是欄數）：

| 頻率 | 表 |
|---|---|
| 日 | `日_K`(15) `日_三大法人`(37) `日_融資券`(24) `日_技術指標`(15) `日_指標`(7) `日_CAPM`(9) |
| 週 | `週_K`(13) `週_大小股東持股`(47) `週_技術指標`(16) |
| 月 | `月_K`(13) `月_營收`(7) `月_技術指標`(16) |
| 季/年 | `季財報_損益表_千`(28) `季財報_資產負債表_千`(22) `季財報_現金流表_千`(29) `季財報_財務指標`(31) `季股利資訊`(12) |
| 大盤 | `TSE` `OTC`（各 24 欄，**單欄矩陣**，與個股矩陣運算會自動廣播）＋ `_週` `_月` 版本 |
| 其他 | `其他_大盤`(12，**欄是 29 個法人分類不是股票**) `其他_籌碼集中`(37，12 個時間窗) |
| 期貨 | **`_fut_TXF`**（底線開頭；1 分 K、257 萬列、2017 起 9.3 年、`settlement` 標合約月） |
| 中繼 | `INFO`（股票代號/名稱/市場/ETF/上市日） `list`（8 個股池） `INDEX`（各頻率日期序列） |

**台股表格 `api.Data`，80 張**，按用途分：清單與狀態 10、盤中大盤概況 5、
全市場排行 12、單檔多期 K線/技術指標 7、單檔多期財報 9、單檔多期籌碼 11、
券商分點 3、分K與逐筆明細 15、批次盤中 3、單值查詢 5。
完整表名見 [data-get-table.md](https://snowyowlstrategy.github.io/data-get-table/)。

**美股**：`api.USMSMP` 10 張（`日_K`(8) `週_K`(5) `月_K`(5) `季營收`(5)
`季股利資訊`(4) ＋ 5 張季財報）、`api.USData` 25 張。

**沒有的**：選擇權、匯率、央行/聯準會利率、原物料、海外期貨。
表裡的「匯率」是財報的「匯率變動對現金之影響」、「利率」是殖利率/毛利率——都不是行情。

### 資料源地圖：問什麼 → 用什麼

**答不出來通常不是沒資料，是找錯地方。** 這張表是查過的：

| 問的是 | 用 |
|---|---|
| 股價（日／週／月） | `MSMP.日_K` / `週_K` / `月_K`。要還原權息用 `adj_*` 欄 |
| 漲跌幅、誰漲停 | `MSMP.日_K.漲跌幅` |
| 成交量／成交額排行 | `MSMP.日_K.成交額_百萬`（或 `Volume`、`市值_百萬`） |
| 三大法人、外資持股比例 | `MSMP.日_三大法人`（**37 欄**：外資／投信／自營的買賣超張數與金額、持股張數、發行總張數、持股成本） |
| 八大行庫（官股）買賣超 | `MSMP.日_三大法人.八大行庫_買賣超張數`，或 `Data.get('查詢八大行庫買賣超-單檔股票多個區間', s)` |
| 融資融券 | `MSMP.日_融資券`（24 欄，欄名帶單位：`融資餘額_張`、`融券餘額_張`、`借券賣出餘額_張`…**不是** `融資餘額`） |
| 券商分點 | `Data.get('查詢近100日買賣張數加總_高至低排序', s)` 拿分點清單，再用 `查詢近100日該分點資料` / `查詢TOP15主力分點買賣超加總資料(指定起訖日)` |
| 大戶／散戶持股結構 | `MSMP.週_大小股東持股`（47 欄，`張數小於1_比例`、`張數1到5_比例`… 分級） |
| 籌碼集中度、主力成本 | `MSMP.日_三大法人.主力買賣超張數` / `主力成本_月均`，或 `Data.get('籌碼集中渙散/主力買賣超-單檔股票多個區間', s)` |
| 月營收 | `MSMP.月_營收`（`金額_千`、`MoM`、`YoY`、`累積YoY`、`營收創歷史新高`），**199 期回到 2010** |
| 季財報、PER、EPS | `Data.get('個股季財報資訊-單檔股票多個區間', s)`（**135 欄、46 期、11.5 年**，含 `本益比`） |
| 股利 | `Data.get('個股股利資訊(年)-單檔股票多個區間', s)`（15 年）或 `(季、含填息天數)` |
| 技術指標 | `MSMP.日_技術指標`（`macd_dif`/`macd_dea`/`macd_hist`、`rsi_5`/`6`/`10`、`K`、`D`、`bulin_upper`/`lower`、`ADX`、`plus_di`/`minus_di`） |
| 大盤指數 | `MSMP.TSE.Close` / `MSMP.OTC.Close`（**單欄矩陣**，與個股矩陣運算時會自動廣播） |
| 大盤法人買賣 | `Data.get('大盤-法人買賣資料-多檔股票最新時間')`（29 列是**法人分類**不是股票） |
| 產業分類與成分股 | `Data.get('台股產業對照表')` 取 `指數彙編分類`（172 種），再 `Data.get('產業個股清單', 那個值)` |
| **台指期** | **`MSMP._fut_TXF`**（1 分 K，**257 萬列、2017 起 9.3 年**，`settlement` 標記合約月）。K 線用 `api.bt.Tools.get_fut_ohlc('TXF', freq=60)` |
| 盤中分K／逐筆明細 | `Data.get('取得歷史1分K(指定起訖日)', s, start, end)`、`取得今日即時成交明細(含五檔報價)`、`取得零股成交明細` |
| 美股即時快照 | `api.USData.get_snapshots(['AAPL'])`（僅美股；台股無此方法，用分K查詢） |
| **條件組合選股** | 積木 `api.Block`（659 個） |
| **回測** | `api.bt`，見 [backtest.md](https://snowyowlstrategy.github.io/backtest/) 起五頁 |

`日_K` 的 15 欄：`處置股`、`Open`、`High`、`Low`、`Close`、`Volume`、`當沖量`、
`成交額_百萬`、`市值_百萬`、`adj_*` 五欄、`漲跌幅`。完整表列見 [msmp-tables.md](https://snowyowlstrategy.github.io/msmp-tables/)；
台股 `api.Data` 共 **80 張**表，見 [data-get-table.md](https://snowyowlstrategy.github.io/data-get-table/)。

### 三條硬規則

**1. 頻率決定歷史長度。日頻只有 1000 筆（約 4.1 年）。**

| 問「近 N 年」 | 走哪 |
|---|---|
| 一週 ~ 四年 | 日頻（1000 筆剛好蓋到 4.1 年） |
| **近五年以上** | **日頻不夠**，降到週頻（`Data` 的還原週K，14.6 年）或季頻（季財報，11.5 年） |
| 近十年以上 | 月營收（16.6 年）、年股利（15 年）、`Data` 還原月K（14.6 年） |

同一個東西 `Data` 版本常比 `MSMP` 版本長：週K 754 筆 vs 604 筆、月K 176 vs 141。
完整對照見 [data-history.md](https://snowyowlstrategy.github.io/data-history/)。

**2. 單位。算錯 100 倍很常見。**

| 欄名結尾 | 換成億元 |
|---|---|
| `_千`（如 `月_營收.金額_千`） | `÷ 1e5` |
| `_百萬`（如 `成交額_百萬`） | `÷ 1e4` |

成交量單位：**整股是張、零股是股、期貨是口**。台股一張 = 1000 股。

**3. 沒有的資料要直接說，不要硬湊。**

snowyowl **沒有**：選擇權、匯率（FX）、央行／聯準會利率、公債殖利率、原物料、海外期貨。
表裡出現的「匯率」是財報的「匯率變動對現金之影響」、「利率」是「殖利率／毛利率／週轉率」
這類財務比率，**都不是行情資料**。

### 實測配方

```python
import snowyowl as so
api = so.login('user@x.com', 'pwd')         # 或 so.login(type='google')
M = api.MSMP
k = M.日_K                                  # index 日期，columns 股票代號

# 今天漲幅 5% 以上 —— 89 檔（資料日 2026-08-20）
t = k.漲跌幅.iloc[-1].dropna()
t[t >= 5].sort_values(ascending=False)

# 今天漲停（>= 9.9%）—— 21 檔
t[t >= 9.9]

# 成交金額前 5 名（百萬元）
k.成交額_百萬.iloc[-1].dropna().nlargest(5)
# 2454 49435 / 2408 45328 / 2330 40117 / 2344 26426 / 2327 24263

# 某檔最近一個月
k.Close['2330'].dropna().tail(20)           # 2350.0 -> 2375.0，+1.06%

# 三大法人近一週（單位：張）
C = M.日_三大法人
import pandas as pd
pd.DataFrame({z: getattr(C, z)['2330'] for z in
              ['外資買賣超張數','投信買賣超張數','自營買賣超張數',
               '三大法人買賣超_張數','八大行庫_買賣超張數']}).tail(5)

# 外資持股比例
(C.持股張數_外資['2330'] / C.發行總張數['2330'] * 100).iloc[-1]      # 69.16

# 月營收（金額_千 ÷ 1e5 = 億元）
r = M.月_營收
r.金額_千['2330'].iloc[-1] / 1e5            # 4676 億（2026-M07）
r.YoY['2330'].iloc[-1]                       # +44.69%

# 近五年 PER（季頻，日頻不夠長）
d = api.Data.get('個股季財報資訊-單檔股票多個區間', '2330')
d[d['年季'] >= '2021-Q1'][['年季','本益比','公告EPS(元)']]     # 22 期，13.1 ~ 32.4

# 台指期近月合約日線
g = M._fut_TXF.set_index('DateTime')
g.resample('D').agg(開盤=('Open','first'), 最高=('High','max'), 最低=('Low','min'),
                    收盤=('Close','last'), 成交量=('TotalVolume','sum'),
                    結算日=('settlement','last')).dropna(subset=['開盤']).tail(5)

# 補股票名稱
name = M.INFO.set_index('股票代號')['股票名稱']
```

> **回答時要講資料日期，並排除雜訊。** `api.MSMP` 只到**前一交易日**，
> 使用者說「今天」通常指最近一個交易日——用 `k.Close.index[-1]` 取出來明講。
> 漲幅排行的前幾名常是興櫃／處置股（成交額只有幾百萬），
> 加個 `成交額_百萬` 門檻或標註出來，答案才有用。

### 積木名稱不是規格，`r['code']` 才是

實例：`漲跌幅介於P1%到P2%之間`（`o6`）聽起來像當日漲跌幅，實際產生的是

```python
close_shifted = close.shift(20)             # ← 20 天，硬寫死、沒有 period 參數
change_pct = ((close / close_shifted) - 1) * 100
```

也就是**近 20 日**漲跌幅。用它問「今天漲 5~9%」會拿到 183 檔，
而當日真正在 5~9% 的只有 49 檔。

而且它的 `percent1` 是**上界**、`percent2` 是**下界**（`percent1 > percent2`，
寫反會 `ValueError`）——與名稱「P1% 到 P2%」的直覺相反。

**結果檔數不符預期時，第一件事是 `print(r['code'])` 看它到底算什麼。**

## 主線四步

```python
# 1. 查這個積木能填什麼（輸出的 'id' 就是條件 dict 的 id）
B.get_table_tw()['今日收盤價<>開高低P%']
# {'id': 'o1', 'operator': ['<','>'], 'ohlc_field': ['adj_Open','adj_High','adj_Low'], 'percent': [1,2,3,5,7,9]}

# 2. 組積木——把所有參數明確寫出來
b1 = B.con_tw('今日收盤價<>開高低P%', operator='>', ohlc_field='adj_High', percent=3)
# {'id': 'o1', 'operator': '>', 'ohlc_field': 'adj_High', 'percent': 3, 'close_field': 'adj_Close', 'period': 1}
#                                                          尾端兩個是規格固定值，con 自動補

# 積木也可以用 id 指定，三種寫法輸出相同：
#   B.con_tw('o1', ...)  /  B.con_tw(id='o1', ...)  /  B.con_tw({'id':'o1', ...})

# 3. 執行（多個積木之間是 AND）
r = B.screener_tw([b1])

# 4. 取用
r['dataframe']   # 通過的股票 + 各積木中間值（stock_id / sig / cond1_*）
r['date']        # 這批結果的資料日期
r['sig']         # 完整訊號矩陣 日期 × 股票代號 的 bool
r['code']        # 等價可執行 python，作用域只要有 api 就能跑（自帶 msmp= 與 helper 定義）
```

美股同名結尾 `_us`。美股**沒有** `pool` 參數，傳了會 TypeError。

## 通用積木：任何條件先用這三顆組（先看這裡）

**使用者講的任何條件，先用這三顆組，不要先去翻 402 顆原生積木。**
實測 30 多個常見條件（KD／MACD／RSI／布林／ADX／K棒型態／財報／營收／排行）**全部組得出來**。
原生積木把窗格、聚合、位移、倍數寫死，八成的自訂需求都卡在合法值清單上。

### 第一步：有哪些資料可以填（`left` / `right` 就填這些）

格式一律 **`'表.欄'`** 單一字串。

| 要什麼 | 填什麼 |
|---|---|
| 價量 | `日_K.Open` `.High` `.Low` `.Close` `.Volume` `.漲跌幅` `.成交額_百萬` `.市值_百萬` `.adj_Close`（還原價） |
| **技術指標**（不用自己算） | `日_技術指標.K` `.D` `.rsi_5` `.rsi_6` `.rsi_10` `.macd_dif` `.macd_dea` `.macd_hist` `.bulin_upper` `.bulin_lower` `.bulin_sma` `.plus_di` `.minus_di` `.ADX` |
| 三大法人 | `日_三大法人.外資買賣超張數` `.投信買賣超張數` `.自營買賣超張數` `.三大法人買賣超_張數` `.主力買賣超張數` `.持股張數_外資` |
| 估值 | `日_指標.本益比` `.股價淨值比` `.市值` `.股本_百萬` `.peg` |
| 融資券 | `日_融資券.融資餘額_張` `.融券餘額_張` `.券資比` `.融資使用率` |
| 籌碼集中 | `其他_籌碼集中.籌碼集中度_5日` `_20日` `_60日`、`.主力買賣超張數_5日`、`.家數差_20日` |
| 月營收 | `月_營收.YoY` `.MoM` `.金額_千` `.累積YoY` |
| 季財報 | `季財報_財務指標.營業毛利率` `.營業利益率` `.稅後淨利率` `.ROE_稅後權益報酬率` `.ROA_稅後資產報酬率` `.每股稅後淨利` `.每股淨值` `.負債比率` `.流動比率` `.營收成長率` `.eps成長率` |
| 股利 | `季股利資訊.年殖利率` `.現金股利` `.股利合計` `.填權息天數`（年頻）、`.現金股利_季`（季頻，依盈餘所屬季；年配息記在 Q4） |
| 單季 EPS | `季財報_損益表_千.eps` |
| 週/月頻 | `週_K.*` `月_K.*` `週_技術指標.*` `月_技術指標.*`（欄位同日頻） |

**美股**（`con_us`）：`日_K.Open/High/Low/Close/Volume`、`季財報_財務指標.*`、
`季營收.YoY`、`季股利資訊.*`。美股**沒有** `漲跌幅`、`adj_*`、`日_技術指標`、`日_三大法人`。

完整欄位清單：[msmp-tables.md](https://snowyowlstrategy.github.io/msmp-tables/)（台股 25 張表、美股 10 張）。

### 第二步：三顆怎麼填

```python
N  = 'dataA<>dataB+C'                  # a1：左 <> 右 + 常數
N2 = '(dataA-dataB)/abs(dataB)<>C'     # a2：(左 - 右) / |右| <> 常數（變化率）
N3 = 'dataA/abs(dataB)<>C'             # a3：左 / |右| <> 常數（比值）
```

N2/N3 的**分母取絕對值**，分母為 0 視為不成立；**常數是比值**（`0.2` = 20%）。
舊名稱 `通用:兩側比較…`／`通用:兩側相減…`／`通用:兩側相除…` 已不能用；`con_tw(id='a1', ...)` 照常可用。

`left` 與 `operator` **必填**，其餘有預設。左右兩側對稱。

| 參數 | 意思 | 預設 |
|---|---|---|
| `left` / `right` | `'表.欄'`。**`right` 不寫 = 純門檻**（只跟 const 比） | left 必填 |
| `operator` | `>` `<` `>=` `<=` `==` `crossup` `crossdown` | **必填** |
| `*_rolling` | 窗格。**1 = 不聚合取原值**；資料不足 N 根時用現有根數算 | 1 |
| `*_agg` | `mean` `max` `min` `sum` `std` `last` | mean |
| `*_shift` | 往前幾根。**1 = 前一根，不含當根** | 0 |
| `*_pct` | 百分比。`35` = ×1.35、`-20` = ×0.8 | 0 |
| `const` | N 加在右側；N2/N3 是門檻（**比值**） | 0 |
| `continue_days` | 窗格 K：連續成立幾根 | 1 |
| `continue_times` | 近 K 根中**至少 M 根**成立；不給 = K；M > K 時恆不成立 | None |
| `rank` | 取前 N 名（`>` 取大、`<` 取小） | None |
| `pool` | **rank 的母體股池**，要和 screener 的 pool 一致 | `['all']` |

單側算式：`agg(data, rolling).shift(shift) * (1 + pct/100)`

`get_table_tw()` 取回來直接填幾格再丟 `con_tw()` 也可以：沒填的 `[]` 用預設值，`left_agg`/`right_agg` 還是整份清單時當 `'mean'`。

### 第三步：翻譯（關鍵詞 → 參數）

| 關鍵詞 | 怎麼填 |
|---|---|
| 「昨日」「前一根」「第二根」 | `shift=1` |
| 「最新一根」「今日」「當根」 | 不寫 shift |
| 「N 日均」 | `rolling=N, agg='mean'` |
| 「N 日最高/最低」 | `rolling=N, agg='max'` / `'min'` |
| 「N 倍」 | `pct=(N-1)*100`，如 1.35 倍 → `pct=35` |
| 「連續 N 天」 | `continue_days=N` |
| 「N 天中至少 M 天」「N 季中至少 M 季」 | `continue_days=N, continue_times=M` |
| 「近四季合計」 | `rolling=4, agg='sum'` |
| 「成長 X%」「較上季增加 X%」 | `N2, const=X/100` |
| 「前 N 名」 | `rank=N` + `pool` |
| 「突破」「向上穿越」 | `operator='crossup'` |
| 「排除 X < N」 | 寫成 `operator='>', const=N`（積木是留下符合的） |
| 「為 0」「沒有動作」 | `operator='==', const=0` |

實測對照（資料日 2026-09-24，季報 2026-08-14，pool 一律 `['tse','otc','~etf']`）：

| 使用者說 | 怎麼寫 | 檔數 |
|---|---|---|
| 成交量 > 3000 張 | `N, left='日_K.Volume', operator='>', const=3000` | 232 |
| 昨日成交量 > 3000 張 | 加 `left_shift=1` | 253 |
| 近5日均量 > 2000 張 | `left_rolling=5, left_agg='mean', const=2000` | 384 |
| 收盤價 > 5日均價 | `left='日_K.Close', right='日_K.Close', right_rolling=5, right_agg='mean'` | 792 |
| 量 > 前一根5日均量×1.35 | 同上加 `right_shift=1, right_pct=35` | 276 |
| 量 > 前5日最大量×1.35 | 同上但 `right_agg='max'` | 109 |
| K > 80 | `left='日_技術指標.K', operator='>', const=80` | 145 |
| KD 黃金交叉 | `left='日_技術指標.K', operator='crossup', right='日_技術指標.D'` | 105 |
| MACD 黃金交叉 | `left='日_技術指標.macd_dif', operator='crossup', right='日_技術指標.macd_dea'` | 66 |
| MACD 紅柱連3日遞增 | `N2, left/right='日_技術指標.macd_hist', right_shift=1, const=0, continue_days=3` | 704 |
| 穿越布林上軌 | `left='日_K.Close', operator='crossup', right='日_技術指標.bulin_upper'` | 54 |
| ADX > 25 | `left='日_技術指標.ADX', operator='>', const=25` | 613 |
| 漲幅 > 5% | `left='日_K.漲跌幅', operator='>', const=5` | 56 |
| 連3日上漲 | 同上 `const=0, continue_days=3` | 86 |
| 收紅K | `left='日_K.Close', operator='>', right='日_K.Open'` | 903 |
| 創20日新高 | `left='日_K.Close', right='日_K.Close', right_rolling=20, right_agg='max', right_shift=1` | 166 |
| 5日線 > 20日線 | `left_rolling=5, left_agg='mean', right_rolling=20, right_agg='mean'` | 996 |
| 黃金交叉(5/20) | 同上但 `operator='crossup'` | 106 |
| 振幅 > 3% | `N3, left='日_K.High', right='日_K.Low', const=1.03` | 504 |
| 本益比 < 15 | `left='日_指標.本益比', operator='<', const=15` | 1015 |
| ROE > 15% | `left='季財報_財務指標.ROE_稅後權益報酬率', operator='>', const=15` | 51 |
| 營收 YoY > 20% | `left='月_營收.YoY', operator='>', const=20` | 853 |
| 營收連3月 YoY > 0 | 同上 `const=0, continue_days=3` | 1057 |
| 營收創12月新高 | `left='月_營收.金額_千', right='月_營收.金額_千', right_rolling=12, right_agg='max', right_shift=1` | 326 |
| 毛利率比上季成長 | `N2, left/right='季財報_財務指標.營業毛利率', right_shift=1, const=0` | 988 |
| EPS 季增 | `N2, left/right='季財報_財務指標.每股稅後淨利', right_shift=1, const=0` | 1231 |
| 單季 EPS 較上季成長 20% | `N2, left/right='季財報_損益表_千.eps', right_shift=1, const=0.2` | 947 |
| 近四季 EPS > 0 | `left='季財報_損益表_千.eps', left_rolling=4, left_agg='sum', const=0` | 1537 |
| 前8季至少6季 EPS > 0 | `left='季財報_損益表_千.eps', const=0, continue_days=8, continue_times=6` | 1293 |
| 投信連續20天沒動作 | `left='日_三大法人.投信買賣超張數', operator='==', const=0, continue_days=20` | 1449 |
| 前2~21天沒動作 | 同上加 `left_shift=1` | 1468 |
| 近10日至少7日投信買超 | `left='日_三大法人.投信買賣超張數', operator='>', const=0, continue_days=10, continue_times=7` | 93 |
| 投信買超佔成交量>5% | `N3, left='日_三大法人.投信買賣超張數', right='日_K.Volume', const=0.05` | 50 |
| 三大法人買超前30名 | `left='日_三大法人.三大法人買賣超_張數', const=300, rank=30, pool=[...]` | 30 |
| 成交額前100大 | `left='日_K.成交額_百萬', const=0, rank=100, pool=[...]` | 100 |
| 股價最低20檔 | `left='日_K.Close', operator='<', const=9999, rank=20, pool=[...]` | 20 |

### 六個翻譯時最容易錯的地方

**1. 「排除 X < N」要寫成 `> N`。** 積木是「留下符合的」不是「排除符合的」。
實測寫反：1602 檔（低量股）vs 正確 384 檔。

**2. 「為 0」是 `==` 不是 `<`。** `<` 是賣超、`>` 是買超。
「連續20天買賣超為0」寫成 `<` 變成「連續20天都賣超」，實測 **0 檔**（正確 1468 檔）。

**3. `rank` 的 `pool` 要和 `screener_tw(pool=...)` 填一樣。**
rank 是橫斷面排名，母體不同結果就不同。只在 screener 給而 con 沒給，
會先在全市場排完前 30 再被切掉，最後不足 30 檔（實測剩 14）。

**4. 月/季/年頻不用自己處理。** `季財報_*`、`月_營收.*`、`季股利資訊.*` 的
index 是字串（`'2026-Q2'`），積木內部已轉成公告日並遮蔽下市資料，直接餵 screener 不會炸。

**5. N2/N3 的常數是比值。** 「成長 20%」寫 `const=0.2`，不是 `20`。

**6. `季股利資訊.年殖利率` 的今年列目前多為 0。** 2026-Y 一公告就蓋過 2025 年的值，
「殖利率 > 5%」現在只剩個位數檔。這是資料源狀態；要用去年殖利率加 `left_shift=1`。

### 什麼時候才回頭找原生積木

MSMP 沒有現成欄位、而且 rolling / shift / continue 組不出來的語意：
「連續N年配息」「葛林布雷排名」「分點籌碼」「產業分類」，以及需要「年度對年度」對齊的比值。
「近四季累計」（`rolling=4, agg='sum'`）、「最近N期至少有M期…」（`continue_times`）、
「填權息天數」現在都能用通用積木寫。
用 `[n for n in B.get_table_tw() if '關鍵字' in n]` 找。

### 完整範例

```python
N = 'dataA<>dataB+C'
POOL = ['tse', 'otc', '~etf', '~KY', '~DR']

# 量能強：昨日量 >3000 張，且最新一根量 > 第二根 5 日均量 ×1.35
b1 = B.con_tw(N, left='日_K.Volume', left_shift=1, operator='>', const=3000)
b2 = B.con_tw(N, left='日_K.Volume', operator='>',
              right='日_K.Volume', right_rolling=5, right_agg='mean',
              right_shift=1, right_pct=35)
B.screener_tw([b1, b2], pool=['tse','otc','~etf'])          # 32 檔

# 外資關注：近5日均量 >2000，前 2~21 天投信皆為 0，當日投信買超 >50
c1 = B.con_tw(N, left='日_K.Volume', left_rolling=5, left_agg='mean',
              operator='>', const=2000)
c2 = B.con_tw(N, left='日_三大法人.投信買賣超張數', left_shift=1,
              operator='==', const=0, continue_days=20)
c3 = B.con_tw(N, left='日_三大法人.投信買賣超張數', operator='>', const=50)
B.screener_tw([c1, c2, c3], pool=POOL)   # 當日 0 檔，全期 460 次 / 262 日

# 三大法人買超 Top30
d1 = B.con_tw(N, left='日_三大法人.三大法人買賣超_張數',
              operator='>', const=300, rank=30, pool=POOL)
B.screener_tw([d1], pool=POOL)                              # 30 檔
```

美股同名用 `con_us()`，但**沒有 `pool` 參數**。細節見 [block-auto.md](https://snowyowlstrategy.github.io/block-auto/)。

## 四條鐵則

**1. 一律用 `con_*()` 組積木，不要自己手寫條件 dict。**
`con()` 會補齊預設值與**規格固定值**、驗證合法值、擋掉打錯的參數名。手寫 dict 漏掉
規格固定值不會報錯，但會靜靜拿到 class 預設值——實測 8 個積木兩者不同，
例如 `o1` 漏掉 `period=1` 會從 595 檔變成 1779 檔。
唯一例外是通用積木 `a1`/`a2`/`a3` 本身（沒有固定值），手寫 dict 安全。
以通用積木寫成的具名積木（例如 `近四季EPS>0`、`現金股利發放率>N%`）有固定值，一樣要用 `con_*()`。

**2. 判斷「積木可不可用」只看 `get_table_*()` 的 key。**
`get_table_tw()` 回 416 個、`get_table_us()` 回 260 個。台股其中 11 個是以 `a1`/`a3` 寫成的具名積木
（名稱不同、共用同一個 id）。registry 的 class 是 409 / 301 個，其中沒出現在 `get_table` 的
在 Excel 標停用或未登錄，`con()` 建不出來，會拋 `找不到積木`。
`list_conditions()` 已**不再掛在 `api.Block`** 上，要用得 `from snowyowl.blocks import list_conditions`。

**3. 零命中時 `dataframe` 是 `(0, 0)`，連欄位都沒有。**

```python
df = r['dataframe']
symbols = df['stock_id'].tolist() if len(df) else []   # 不要無條件取欄，會 KeyError
```

**4. 用 `sig` 的欄數分辨「條件嚴格」與「積木壞了」。**

| `sig` 形狀 | 意思 |
|---|---|
| `(1000, 2865)`，`dataframe` 是 `(0,0)` | 正常，只是當日沒人通過 |
| `(1000, 0)` | 訊號矩陣裡沒有股票 → **積木有問題** |

## 不要用的積木

**這幾個會給出看似合理但語意不同的答案，或直接拋錯。遇到需求時不要硬套——
用替代路徑，或直接說「目前沒有這個」。**

| 不要用 | 為什麼 | 改用什麼 |
|---|---|---|
| `漲跌幅介於P1%到P2%之間`（`o6`） | 名稱沒提期間，實際算的是**近 20 日**漲跌幅，而且 `20` 寫死改不了。`percent1` 是上界、`percent2` 是下界，與名稱直覺相反 | 當日漲跌幅走 `MSMP.日_K.漲跌幅`；近 N 日走 `Close.pct_change(N)*100` |
| 8 個複合值積木的 `table` / `direction` | `o29`／`o30`／`v5`／`v7`／`v8`／`m5`／`m7` 的 `table` 與 `r4` 的 `direction`，`get_table` 顯示 `['日','周','月']` 但 class 要 `'日_K'`，傳任一選項都 `ValueError` | **不傳這個參數**，用預設值（＝只有日頻）。使用者要週／月頻時**直接說這幾個積木目前只支援日頻** |

> **原則：寧可說「沒有這方面的資料」，也不要用語意不符的積木去湊答案。**
> 拋錯是大聲的、看得見；語意不符的答案是安靜的，使用者會當真。

## 積木的其他規則

**`con()` 驗規格，`screener()` 驗 class——兩層。**
`con()` 只比對合法值清單；class 自己的限制（像上面那個 `table`）要到 `screener()` 才浮現。
**`con()` 過了不代表能跑**，組完要真跑一次。

**事後改 dict 不會被驗證。** `c['top_n']=5` 不合法也照跑。一律用 `con(name, **params)`。

**規格釘死的參數不可指定。** 台股 28 個、美股 12 個積木有固定值
（例如 `D日均線斜率<>0.4` 的 `value`），`con()` 會自動帶出來，你再傳**不同的值**會拋錯
（傳相同的值放行，所以 `con(con(x))` 成立，可以拿既有條件改參數）。

**`category` 有兩套詞彙。** `list_conditions(category=..., market=...)` 只吃英文模組名
（台股 11 種：`ohlc`/`volume`/`financial`/`technical`/`ranking`/`moving_average`/
`sanda`/`margin_trading`/`dahu`/`industry`/`chips`）。中文次類別只用於文件分組，
傳進去回空 dict。

**兩組積木是別名，不是 bug。** 台股 `m21` 的 `(old)`、美股 `r23` 的「成交值/成交金額排行」
——描述同一件事，跑出同一結果就是對的。

## 股池（只有台股）

```python
B.screener_tw([b1], pool=['all', '~etf'])     # ~ 前綴是排除
```

`all` 2873 / `tse` 1125 / `otc` 923 / `esm` 412 / `etf` 382 / `KY` 143 / `DR` 14 / `delist` 145。
多個不帶 `~` 的取聯集。預設 `['all']`。

> `api.MSMP.list['all']` 含一個重複代號（`'7834'`）。用 `isin()` 篩不受影響，但**不要拿它當 DataFrame 的 columns**，否則後續 reindex 會 `InvalidIndexError`；需要時先 `list(dict.fromkeys(...))`。

## 找積木

積木名稱是中文，`get_table_*()` 的 key 就是名稱，直接字串比對：

```python
[n for n in B.get_table_tw() if '均線' in n]
```

完整清單含每個積木的 id、邏輯模板、詳細說明、合法值與預設值：
<https://snowyowlstrategy.github.io/block-reference-tw/>（405 個，依中文次類別分 37 組）、[block-reference-us.md](https://snowyowlstrategy.github.io/block-reference-us/)（260 個）。

**找不到、或找到的積木參數合法值不合用**（要的數字不在清單裡、要比前一根而積木只比當根、
要連續 N 天而積木寫死 3 天），改用[通用積木](#通用積木既有積木表達不了時用這三顆)那三顆，
不要自己拿 MSMP 算——自己算就不是積木，`r['code']` 也產不出來。

## 資料日期

`api.MSMP` 只到**前一交易日**；美股 `api.USMSMP` 通常比台股再晚一天。所以 `screener_tw` 與
`screener_us` 的 `date` 不會相同，不要假設一致。要當日盤中資料用 `api.Data.get()` 的分K。季頻／月頻積木的 index 可能排到未來（財報預定公布日），此時 `date='last'` 會取到未來日期。

## 美股

積木 260 個，方法都是 `_us` 結尾（`con_us` / `screener_us` / `get_table_us`）。
資料在 `api.USMSMP`（矩陣）與 `api.USData`（表格，25 張）。

**四個跟台股不一樣、會讓你答錯的地方：**

**1. 一定要加流動性門檻，而且成交額要自己算。**
美股 13174 檔裡有大量權證、次分錢股與反分割假象。實測「今天漲幅 ≥10%」
不加門檻是 **369 檔**，第一名 `NRDY +1444%` 其實是 0.65→10.10 的反分割
（當天只成交 5.3 萬股／50 萬美元）。**美股沒有 `成交額_百萬` 欄**，要自己乘：

```python
U = api.USMSMP
c, v = U.日_K.Close, U.日_K.Volume
val = (v.iloc[-1] * c.iloc[-1]).dropna()          # 成交值（美元）
chg = c.pct_change().iloc[-1].dropna() * 100      # 美股也沒有 漲跌幅 欄
big = chg[(chg >= 10) & (val.reindex(chg.index).fillna(0) >= 1e8)]
```

門檻效果：`369 檔 → 222`（≥$1M）→ `138`（≥$10M）→ **`55`**（≥$100M）。
加門檻後前幾名才是真的：`MRNX +352%`、`MRNA +177%`（成交值 347 億美元）。

**2. 沒有 `漲跌幅`、沒有 `adj_*`。** 漲跌幅用 `Close.pct_change()*100` 自己算。
`日_K` 只有 8 欄：`Open/High/Low/Close/Volume/買量/賣量/資金淨流`。

**3. 「降頻拿長歷史」不成立。** 台股週／月 K 有 11.7 年，
**美股週 K 只有 4.5 年、月 K 只有 4 年**——跟日 K 差不多。
美股的長歷史只在**財報**（`USMSMP.季財報_*`，111 期回到 1998-Q4）
與**股利**（34 期回到 1993）。要長期股價：沒有。

**4. 沒有 `pool`。** `screener_us(blocks, pool=[...])` 會 `TypeError`。
要限定範圍自己切 columns，或先用 `USData.get('美股產業對照表')`
（13187 列，含 `粗產業名稱`／`細產業名稱`／`是否為ETF`／`市值(美元)`）取代號清單。

> 美股資料日通常**比台股晚一天**（實測台股 2026-08-20 / 美股 2026-08-19），
> 財報覆蓋 5801 檔而價格覆蓋 13174 檔——跨表運算後要 `.dropna()`。
> 細節見 [usmsmp.md](https://snowyowlstrategy.github.io/usmsmp/)、[usdata.md](https://snowyowlstrategy.github.io/usdata/)。

## 盤中快照（當下的價格）

`api.MSMP` 只到前一交易日。要當下的價格：

- **美股**：`api.USData.get_snapshots(['AAPL'])` → `dict[str, Symbol]`（open/high/low/close/volume/previousClose/netChange/percentChange）
- **台股**：沒有 get_snapshots；用 `api.Data.get(...)` 的分K／逐筆歷史查詢，見 [intraday.md](https://snowyowlstrategy.github.io/intraday/)

> **本版不含即時行情訂閱（推送）。** `api` 上沒有 `SWQuote` / `Quote` /
> `USQuote` / `FutQuote`。當下資料只有：**美股** `api.USData.get_snapshots()`（快照），
> 台股用 `api.Data.get()` 的分K／逐筆歷史查詢（台股無 `get_snapshots`）。

## 積木 sig → 策略：部位大小怎麼給

**積木最主要的用途是拿 `sig` 去搭 backtest_plus 建策略。**
`sig` 是「日期 × 股票代號」的 bool 矩陣，一檔股票一個 `Core`，
`generate_report` 收 `{config: core}` 把它們併起來。

部位大小看策略性質，兩種寫法：

### A. 換股／輪動 —— `qty = 1/n`

n 檔之間輪流持有、共用一份資金，每次進場投入資金池的 `1/n`：

```python
N = len(SYMS)
self.Condition.Global(True, 1/N, 0, self.PriceType.market_ioc, None, 'bk')
```

`qty < 1` 是**佔資金池比例**，這時 `capital` 才真的生效，
`rp.report['Strategy']` 才是共用資金池的組合視角。

> 實測 3 檔、`capital=1000 萬`、`qty=1/3`：單筆進場中位 **270 萬**、
> `MaxPosition` 856 萬（< capital，沒有超額）、2 筆因資金不足被夾成 0 張。

**`capital` 只扣已實現損益、不扣未平倉佔用**，所以 `qty × 同時持有檔數 > 1`
就是隱性槓桿——取 `1/n` 剛好是安全上限。

### B. 各檔訊號獨立 —— `qty = 固定金額 / price`（以股計）

每檔各進各的、互不排擠，每筆固定名目金額。`qty` 就是股數，
**`config.unit` 預設就是 `1`（以股計），不用動它**：

```python
cfg = bt.config(symbol=s, fee=0.001425, tax_short=0.003)   # unit 用預設 1
...
def on_bar(self, row):
    if row['sig'] and self.global_qty == 0:
        qty = 100000 / row['Close']                        # 每筆 10 萬元
        self.Condition.Global(True, qty, 0, self.PriceType.market_ioc, None, 'bk')
```

> 實測：單筆進場金額中位 **100,077**，正好 10 萬。

**只有刻意設 `unit=1000`（以張計）時才要注意**——那時 `qty` 被當成**張數**，
同一條公式會變成每筆 1 億：

| 寫法 | 單筆進場金額 |
|---|---|
| `qty = 100000/price`，`unit` 用預設 1（股） | **100,077** ← 對 |
| `qty = 100000/price`，`unit=1000`（張） | **100,076,656** ← 差 1000 倍 |

以張計要自己換算：`qty = max(1, round(金額 / (price * 1000)))`。
但台股一張動輒幾十萬，10 萬的目標在張制下根本買不到一張——
**要精準控制名目金額就用預設的 `unit=1`。**

### 兩者的差別

| | A. 換股 `1/n` | B. 獨立 `金額/price` |
|---|---|---|
| `qty` | < 1，佔資金池比例 | ≥ 1，實際股數 |
| `unit` | 1000（張）或 1（股）都行 | **必須 1** |
| `capital` | 生效，是資金約束 | 不生效（固定口數走向量化路徑） |
| 適合 | n 檔輪動、資金有限 | 各訊號獨立、看單一規則的期望值 |
| 看哪一欄 | `rp.report['Strategy']` | 逐檔欄位各自看 |

### 用選股結果當標的 = look-ahead

拿今天的選股結果去回測過去，是用結果反選樣本，必然偏樂觀——
只能當訊號行為的觀察，不能當策略績效。

```python
r = B.screener_tw([con], pool=['all', '~etf'])
syms = r['dataframe']['stock_id'].head(5).tolist() if len(r['dataframe']) else []
```

> `r['dataframe']` 是**按代號排序**的，`head(5)` 拿到 `1101/1103/1104…`
> 而不是「最好的 5 檔」。要挑標的請自己排序（按成交額或積木的中間值欄）。

要評估規則本身，改用一組**與選股結果無關**的標的（例如固定的權值股）。

> 實測那 5 檔（capital 1000 萬、`qty=1/5`）：TotalReturns -39.8%、263 筆交易、
> 成本 258 萬，而扣成本前的毛損益只有 -186,300——**虧的幾乎全是成本**。

## 單檔技術指標回測：照抄就能跑的骨架

**要寫回測先看這節，不要從六頁文件拼。** 這份是實跑驗證過的完整腳本
（0050 MA5/MA10、每次 10 萬），換標的與訊號就能重用。

```python
import sys
sys.path.insert(0, r'<snowyowl_project 路徑>')      # 用原始碼樹時才需要
import pandas as pd
import snowyowl as so

api = so.login(type='google')          # 或 so.login('user@x.com', 'pwd')
bt = api.bt
SYM, AMOUNT = '0050', 100_000

# ── 1) 日 K 從 MSMP 取。要 adjclose 就整組都用 adj_*（含 adj_Volume）────
k = api.MSMP.日_K
df = pd.DataFrame({
    '開盤價': k.adj_Open[SYM], '最高價': k.adj_High[SYM],
    '最低價': k.adj_Low[SYM],  '收盤價': k.adj_Close[SYM],
    '成交量': k.adj_Volume[SYM],
}).dropna()                            # ← 一定要 dropna，未上市期間是 NaN
df.index.name = 'DateTime'
df = df.reset_index()

# ── 2) 策略 ────────────────────────────────────────────────────
class MaCross(bt.Core):
    def on_init(self):                 # 只放參數與狀態，下單會 RuntimeError
        self.n_fast, self.n_slow = 5, 10
        self.amount = AMOUNT
        self.close_list = []

    def on_bar(self, row):
        self.close_list.append(row['Close'])          # row 的 key 一律英文
        fast = self.Ind.rolling_mean(self.close_list, self.n_fast)
        slow = self.Ind.rolling_mean(self.close_list, self.n_slow)
        self.char['ma5'], self.char['ma10'] = fast[-1], slow[-1]   # 進 char 才上圖（show_monitor 與 dashboard 的 K 線）
        if len(self.close_list) < self.n_slow:
            return                                     # 暖身期不下單

        if fast[-1] > slow[-1] and self.global_qty == 0:
            qty = self.amount / row['Close']           # ★ 股數，搭 unit=1
            self.Condition.Global(True, qty, 0, self.PriceType.market_ioc, None, 'bk')
        elif fast[-1] < slow[-1] and self.global_qty > 0:
            self.Condition.Global(True, -self.global_qty, 0,
                                  self.PriceType.market_ioc, None, 'bp')

core = MaCross()
core.feed_data(df, ['DateTime', '開盤價', '最高價', '最低價', '收盤價', '成交量'])

# ── 3) 成本：0050 是 ETF → tax_short=0.001（個股才 0.003）─────────
cfg = bt.config(name='MA5_10_0050', symbol=SYM,
                fee=0.001425, tax_long=0, tax_short=0.001, unit=1)

bench = df.set_index('DateTime')['收盤價']             # buy&hold 對照
rp = bt.generate_report({cfg: core}, capital=1_000_000, benchmark=bench)

# ── 4) 驗證三問（每次回測都要做）───────────────────────────────
COL = 'Strategy'                                       # 單策略時就是這個欄名
td = rp.trade_detail
gross, cost = (td['pnl'] + td['cost']).sum(), td['cost'].sum()
invest = rp.report.loc['Invest', COL]
print(f'毛損益 {gross:,.0f} / 成本 {cost:,.0f}（佔 Invest {cost/invest*100:.1f}%）')
print(f'期間 {rp.report.loc["StartDate",COL]} ~ {rp.report.loc["EndDate",COL]}')
print(f'曝險率 {(rp.daily["position"].abs()>0).mean()*100:.1f}%')

rp.dashboard(path='dashboard.html', view=None)         # 腳本裡要給 path
```

**實測結果**（2022-08-15 ~ 2026-09-11，4.08 年）：總報酬 +81.04%、CAGR 16.31%、
MDD -14.23%、Sharpe 1.38、53 筆交易、成本 20,644（佔 Invest 20.0%）、曝險率 61.3%。
對照 0050 buy&hold +292.84%（同期間）——**總報酬輸，但 MDD 與波動都只有一半。**

### 換成別的需求要改哪裡

| 需求 | 改哪 |
|---|---|
| 未還原價 | `adj_Open/High/Low/Close/Volume` → `Open/High/Low/Close/Volume` |
| 個股不是 ETF | `tax_short=0.001` → `0.003` |
| 用「張」思考 | `unit=1000`，且 `qty` 要自己換算成張數（見下方 B 段） |
| 固定比例而非固定金額 | `qty` 填 0~1（如 `1/5`），`capital` 才會生效 |
| 用積木訊號進出場 | 見下一節，`sig` 欄要在 `feed_data(..., columns=['sig'])` 宣告 |
| 加停損停利 | `on_bar` 裡**無條件**每根呼叫 `self.Condition.set_STL('bk', 2.0)` |
| 期貨 | `on_init` 加 `self.tick_map = [(np.inf, 1)]`，config 用 `fee=500, unit=200` |
| 多空反手 | 拆兩張：先 `Global(True, -g, 0, mkt, False, 'net')` 平倉，再 `Global(True, ±1, 0, mkt, False, 'bk'/'sk')` 開倉（合成一張損益相同，但規則統計會標錯） |
| 多檔輪動、資金小 | **`unit=1` 走零股**，否則 `capital/N` 買不起一張會全被夾成 0 |

**完整的七個配方在 [recipes.md](https://snowyowlstrategy.github.io/recipes/)**：單檔技術指標、積木訊號、多檔組合、
期貨雙均線多空反手、目標權重輪動（含零股）、訊號驗證（IC/分組單調）、參數尋優。
每份都實跑驗證過，要寫回測先看那頁。

### 三個只有實跑才知道的坑

**1. 月頻／季頻的字串 index 用 `index_str_to_date()` 轉，不要自己接。**

```python
rev = api.MSMP.月_營收.YoY          # index: '2026-M08'（營收所屬月份）
rev_d = rev.index_str_to_date()     # index: 2026-09-10（公告日，已對到交易日）
```

它同時做了三件事：月頻→次月 10 日、季頻→財報公告日（Q4→次年 3/21）、
**自動遮蔽下市後的資料**（防倖存者偏差）。直接拿 `'2026-M08'` 對齊日頻就是未來函數。

**2. `corrwith` 會炸，要先 `pd.DataFrame()` 脫殼。**

MSMP 回的是 `ProboDataFrame`，覆寫了 `__mul__` 去做自動 reshape，
pandas 的 `corrwith` 內部做 `ldem * rdem` 時會走進去然後爆
`TypeError: '<=' not supported between 'str' and 'Timestamp'`。
算 IC 這類統計前先 `pd.DataFrame(...)` 脫殼；一般矩陣運算則要保留 ProboDataFrame。

**3. 小資金多檔輪動只能做零股。** `unit=1` 就是以股計價。
實測 capital=100 萬跑 20 檔：零股正常（單筆 53,031 元、最小 120 股），
整股（`lot_size` step=1000）全期只成交 2 筆然後拋 `make_daily` 那個誤導訊息。
**框架沒有最低手續費參數**，無法表達零股 20 元低消，小額的真實成本會比回測高。

## 用積木回測

**關鍵：日 K 也從 MSMP 取，不要用 `bt.Tools.get_stock_ohlc`。**
積木的 `sig` 是日頻（來自 MSMP 日_K），而 `get_stock_ohlc` 是 1 分 K resample 的結果——
頻率對不上，還要多下載一份 1 分 K。兩邊都取自 MSMP 日頻時 index 天然完全對齊。

```python
import pandas as pd

def daily_bars(api, symbol):
    k = api.MSMP.日_K
    zh2en = {'開盤價':'Open','最高價':'High','最低價':'Low','收盤價':'Close','成交量':'Volume'}
    df = pd.DataFrame({zh: getattr(k, en)[symbol] for zh, en in zh2en.items()})
    df.index.name = 'DateTime'
    return df.reset_index()

SYM = '2330'
r = api.Block.screener_tw([api.Block.con_tw('今日收盤價<>D日(周月)均線', period=20, table='日_K')])
bars = daily_bars(api, SYM)
bars['sig'] = bars['DateTime'].map(r['sig'][SYM]).fillna(False)   # index 對齊，不需 ffill

class BlockStrategy(api.bt.Core):
    def on_bar(self, row):
        if row['sig'] and self.global_qty == 0:
            self.Condition.Global(True, 1, 0, self.PriceType.market_ioc, None, 'bk')
        elif not row['sig'] and self.global_qty > 0:
            self.Condition.Global(True, -self.global_qty, 0,
                                  self.PriceType.market_ioc, None, 'bp')

core = BlockStrategy()
core.feed_data(bars, ['DateTime','開盤價','最高價','最低價','收盤價','成交量'],
               columns=['sig'])                    # 額外欄位一定要在 columns 宣告，否則 row 裡沒有
cfg = api.bt.config(name=f'block_{SYM}', symbol=SYM, fee=0.001425,
                    tax_long=0, tax_short=0.003, unit=1000)   # 見下方「成本」段
rp = api.bt.generate_report({cfg: core}, capital=1_000_000)
```

### 報表產出

`rp.report`（28 項 KPI，回撤看 `MddRatio` 比率與 `MddAmount` 金額）、`rp.trade_detail`（逐筆交易）、
`rp.trade_main`（委託）、`rp.position`、`rp.daily`（權益曲線，`dd` 欄是每日回撤金額）、
`rp.dashboard()`（互動 HTML 報表，四個分頁；個股分析的 K 線會畫策略的 `self.char`／`self.char1`）、
`rp.save()`（存成 xlsx：report / daily / trade_main / trade_detail / 個股績效 / 進出場規則績效）。

**沒有 `rp.trade`**——逐筆在 `rp.trade_detail`。欄位定義見 [backtest-report.md](https://snowyowlstrategy.github.io/backtest-report/)。

#### 讀 rp.report 的四件事（實測踩過）

**1. 單策略時欄名是 `Strategy`，不是 `config.name`。**
`name='MA5_10_0050'` 只在**多策略**時才會成為欄名；只放一個 core 時整份報表只有
`Strategy` 一欄（有 benchmark 就多一欄 `benchmark`）。所以
`rp.report.loc['Invest', 'MA5_10_0050']` 會 `KeyError`。安全寫法：

```python
COL = 'Strategy' if len(core_dic) == 1 else cfg.name
invest = rp.report.loc['Invest', COL]
# 或直接 rp.report.columns 看有哪些欄
```

**2. `rp.dashboard()` 在 notebook 以外會寫檔，而且預設寫到「當前工作目錄」。**
底層簽名是 `dashboard(path=None, view='notebook', page_size=20, unit=1e4, unit_label='萬')`
（在 notebook 裡用 `view='notebook'` 又不給 `path` 時只內嵌顯示、不存檔）：

| 參數 | 行為 |
|---|---|
| `path=None`（預設） | 自動命名 `YYYYMMDD_dashboard_N.html` 存到 `Path.cwd()`，`N` 從 1 遞增不覆蓋 |
| `path='out.html'` | 存到你指定的路徑 |
| `view='notebook'`（預設） | Jupyter 內嵌顯示；**非 notebook 環境會印 `<IPython.core.display.HTML object>`** |
| `view='web'` | 存檔後用瀏覽器開啟 |
| `view=None` | 只存檔，不顯示不開啟 |

**在終端／腳本裡要明確給 `path`**，否則檔案會散落在當前目錄：

```python
rp.dashboard(path='dashboard.html', view=None)     # 腳本用這個
```

**3. 曝險率要自己從 `rp.daily` 算。** 這是判斷「輸給 buy&hold 是不是結構性問題」的關鍵指標，
報表 28 項 KPI 裡沒有：

```python
exposure = (rp.daily['position'].abs() > 0).mean()    # 有部位的交易日佔比
```

空手的每一天在多頭市場都是純機會成本。曝險 60% 的策略要贏 buy&hold，
剩下 60% 時間得做出 100% 的成績——通常做不到，這是結構性的，掃參數救不了。

**4. `benchmark` 欄的報酬率從「策略第一次進場日」起算，不是資料第一天。**
`_make_benchmark_report(benchmark, index_list, start_date)` 刻意用策略的 `StartDate`
對齊，讓兩邊比較的是同一段期間。所以：

```python
rp.report.loc['TotalReturns', 'benchmark']      # 2.9284 ← 從 StartDate 起算
bench.iloc[-1] / bench.iloc[0] - 1              # 3.0703 ← 從資料第一天起算
```

**兩個都對，只是區間不同。** 要引用「buy&hold 報酬」時說清楚是哪一個，
不要以為算錯了。暖身期越長（MA240 之類）差距越大。

### 回測會答錯的三件事

**1. `tax_long` / `tax_short` 是進場稅／出場稅，與多空無關。**
成本公式是 `entry_price*(tax_long+fee) + exit_price*(tax_short+fee)`。
台股證交稅只在賣出收，所以**只要填 `tax_short=0.003`**，`tax_long` 保持預設的 0。

| 標的 | 設法 |
|---|---|
| 台股個股（以張計） | `fee=0.001425, tax_short=0.003, unit=1000`，`qty` 是張數 |
| 台股個股（以股計） | `fee=0.001425, tax_short=0.003`，`unit` 用預設 1，`qty` 是股數 |
| 台股 ETF | 同上但 `tax_short=0.001` |
| 台指期 | `fee=500, tax_long=0.00002, tax_short=0.00002, unit=200` |
| 美股 | `fee=0, tax_long=0, tax_short=0, unit=1` |

**2. 固定口數下單時 `capital` 是惰性參數。** 實測 10 萬與 1 億跑出**一模一樣**的
`Invest`／`MaxPosition`／`TotalReturns`。要讓資金池生效，`qty` 要填 0~1 的比例
（`Condition.Global(True, 0.2, ...)` = 投入當下資金池 20%）。
這時 `rp.report['Strategy']` 才是共用資金池的組合視角，其餘各欄是「該檔獨佔全部資金」，
**逐檔加總不等於 Strategy**。而且 `capital` 只扣已實現損益、不扣未平倉佔用，
所以 `weight × 同時持有檔數 > 1` 就是隱性槓桿——取 `1/檔數` 最安全。

**3. 先看 `TradeCost` 再看報酬率。** 日頻訊號策略常常毛損益接近零、虧損全來自成本
（實測單檔 2330：毛 +22,000、成本 325,378）。用 `(pnl + cost).sum()` 看毛損益。

現成的組合回測封裝在 `snowyowl_mcp.backtest`：

```python
from snowyowl_mcp import backtest as BT
rp = BT.run(api, [con], ['2330','2317','2454'], capital=10_000_000)   # weight 自動 1/3
```

**其餘全在 docs**：[backtest-strategy.md](https://snowyowlstrategy.github.io/backtest-strategy/)（事件順序、`row` 欄位、`on_position` 早於 `on_bar`、
完全平倉不觸發）、[backtest-order.md](https://snowyowlstrategy.github.io/backtest-order/)（五種 `PriceType`、12 個成交價實測案例、四個停損停利）、
[backtest-data.md](https://snowyowlstrategy.github.io/backtest-data/)（餵資料與週期）、[backtest-report.md](https://snowyowlstrategy.github.io/backtest-report/)（28 項 KPI 定義）——另有 dashboard 四個分頁與 `rp.save()`、
[backtest-tools.md](https://snowyowlstrategy.github.io/backtest-tools/)（25 個指標與尋優，`optimize` 以 `CalmarRatio` 評分）。

期貨回測另外要記得兩件事：`on_init` 設 `self.tick_map = [(float('inf'), 1)]`（預設是股票跳動，
≥1000 以 5 為單位，限價／停損／`compel_now` 價格都會被修到 5 點），以及結算棒要用 `compel_now` 帶 `row['Close']` 強平。

## 其他命名空間

| | 用途 | 文件 |
|---|---|---|
| `api.Data` / `api.USData` | 表格式查詢（清單、財報、分K），`get(table, *args)` | [data-get.md](https://snowyowlstrategy.github.io/data-get/) |
| `api.MSMP` / `api.USMSMP` | 矩陣式歷史資料 日期 × 股票代號 | [msmp.md](https://snowyowlstrategy.github.io/msmp/)、[msmp-tables.md](https://snowyowlstrategy.github.io/msmp-tables/) |
| `api.bt` | 回測 | [backtest.md](https://snowyowlstrategy.github.io/backtest/) 起共五頁（`-strategy` / `-order` / `-data` / `-report` / `-tools`） |

snowyowl **沒有下單功能**，`api` 上不存在 `Order`。

## 維護

Excel（`bulid_spec/台股積木API.xlsx` / `美股積木API.xlsx`）改動後要重建規格。只需要那兩份 Excel，不登入也不連網：

```bash
cd bulid_spec && python build_spec.py     # 產出 block_spec.json 在同層
```

**產完要複製到 `snowyowl/blocks/block_spec.json`**——程式只讀那一份（`blocks/spec.py` 用 `dirname(__file__)` 定位），沒複製會繼續用舊規格而且不會報錯。腳本結尾會比對兩份並把複製指令印出來。

它會把「標啟用卻缺 python編碼」「積木名稱重複覆蓋」「同 python編碼」「依邏輯基礎校正固定值」
「python參數 無法解析」逐筆列出。

維護用的兩支腳本在**原始樹**，curl 版拿不到——安裝完整性檢查請用 `snowyowl-setup --check`。原始樹裡有：

| 指令 | 查什麼 |
|---|---|
| `python check_docs.py` | **文件宣告 vs API 實際**——表清單、欄位數、積木數、頻率長度是否還對得上。上游改了東西文件不會自己知道，靠這支抓 |
| `check.ipynb` | 積木全量健康檢查（con → 建構 → 產碼 → 選股） |

