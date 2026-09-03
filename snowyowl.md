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
  即時行情：api.SWQuote（StarWave）、api.Quote／USQuote／FutQuote、即時報價、盤中快照、
  五檔、逐筆、K棒訂閱。
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

台股 402 個、美股 257 個積木。**規格的唯一真相來源是 `get_table_*()`**，不是 Excel、不是 `list_conditions`。

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
| 即時報價（推送） | **首選 `api.SWQuote.subscribe()`**（有就用它，台美期統一）；退路 `api.Quote`/`USQuote`/`FutQuote`。見 [swquote.md](https://snowyowlstrategy.github.io/swquote/) |
| 一次性快照 | `api.Data.get_snapshots(['2330'])` |
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
# 1. 查這個積木能填什麼
B.get_table_tw()['今日收盤價<>開高低P%']
# {'operator': ['<','>'], 'close_field': ['adj_Open','adj_High','adj_Low'], 'percent': [1,2,3,5,7,9]}

# 2. 組積木——把所有參數明確寫出來
b1 = B.con_tw('今日收盤價<>開高低P%', operator='>', close_field='adj_High', percent=3)
# {'id': 'o1', 'operator': '>', 'close_field': 'adj_High', 'percent': 3}

# 3. 執行（多個積木之間是 AND）
r = B.screener_tw([b1])

# 4. 取用
r['dataframe']   # 通過的股票 + 各積木中間值（stock_id / sig / cond1_*）
r['date']        # 這批結果的資料日期
r['sig']         # 完整訊號矩陣 日期 × 股票代號 的 bool
r['code']        # 等價可執行 python，只需作用域有 msmp（美股 usmsmp）
```

美股同名結尾 `_us`。美股**沒有** `pool` 參數，傳了會 TypeError。

## 四條鐵則

**1. 一律明確指定所有參數，不要依賴預設值。**
台股 59 個、美股 18 個積木的預設值**不在自己宣告的合法值清單內**（`con(name)` 的輸出再原樣傳回 `con()` 會被拒絕）。有些甚至值域不同——例如 `o1` 預設 `close_field='Close'`（未還原）而合法值是 `adj_*`（還原）。

**2. 判斷「積木可不可用」只看 `get_table_*()` 的 key。**
`list_conditions_tw()` 回 406 個、`get_table_tw()` 回 402 個（美股 298 / 257）。多出來的在 Excel 標停用或未登錄，`con()` 建不出來，會拋 `找不到積木`。

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
（例如 `D日均線斜率<>0.4` 的 `value`），`con()` 會自動帶出來，你再傳會拋錯。

**`category` 有兩套詞彙。** `list_conditions_*(category=...)` 只吃英文模組名
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
<https://snowyowlstrategy.github.io/block-reference-tw/>（402 個，依中文次類別分 37 組）、[block-reference-us.md](https://snowyowlstrategy.github.io/block-reference-us/)（257 個）。

## 資料日期

`api.MSMP` 只到**前一交易日**；美股 `api.USMSMP` 通常比台股再晚一天。所以 `screener_tw` 與
`screener_us` 的 `date` 不會相同，不要假設一致。要當日盤中資料用 `api.Data.get()` 的分K 或
`api.SWQuote`。季頻／月頻積木的 index 可能排到未來（財報預定公布日），此時 `date='last'` 會取到未來日期。

## 美股

積木 257 個，方法都是 `_us` 結尾（`con_us` / `screener_us` / `get_table_us`）。
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

## 即時行情（盤中才有用）

`api.MSMP` 只到前一交易日。要當下的價格有兩條路：

```python
# 一次性快照（簡單，要幾檔就幾檔）
api.Data.get_snapshots(['2330'])       # dict[str, Stock]，15 欄

# 推送訂閱（要維持連線）
api.SWQuote.set_cb(on_quote)           # 一定要先 set_cb 再 subscribe
api.SWQuote.subscribe('2330')          # odd=True 訂零股；期貨要帶月份 'TXFH6'
```

**推送訂閱有兩套：`api.SWQuote`（StarWave）與 `api.Quote` / `api.USQuote` /
`api.FutQuote`（舊路徑，台股／美股／期貨分開）。**

> **優先用 `api.SWQuote`。** 帳戶同時有 `SWQuote` 與 `Quote`/`FutQuote` 時，
> 一律走 `SWQuote` —— 它涵蓋台股、美股、期貨，介面統一。`Quote`/`USQuote`/`FutQuote`
> 是相容用的舊路徑，只在 `SWQuote` 不存在時才退而求其次。
>
> **先用 `hasattr(api, 'SWQuote')` 判斷。** 沒訂閱即時行情的帳戶，`api` 上
> **根本不會掛 `SWQuote` 這個屬性**（不是 None，是不存在）—— 直接 `api.SWQuote`
> 會 `AttributeError`。所以順序是：
>
> **現況（2026-09）**：SWQuote 的開通標記後端還沒開始回，所以目前**所有帳戶
> `hasattr(api,'SWQuote')` 都是 False**，實際都會走到退路 `api.Quote` 家族。
> 等後端上線標記後，有訂閱的帳戶才會自動拿到 `api.SWQuote`。這段邏輯先寫好備用。

```python
if hasattr(api, 'SWQuote'):
    q = api.SWQuote                    # 首選：統一介面，台美期都走這條
    q.set_cb(on_quote)                # 一個 callback 收全部
    q.subscribe('2330')               # 一個 subscribe，帶 odd=True 訂零股
```

退路 `api.Quote` / `api.USQuote` / `api.FutQuote` 的**介面和 SWQuote 不一樣**，
不是一個 `subscribe` 打天下，而是**分成四種通道**，每種各有自己的 `set_cb_*`：

```python
q = api.Quote                         # 台股（美股用 api.USQuote，期貨 api.FutQuote）
q.set_cb_tick(on_tick)                # 通道要成對：set_cb_tick 配 subscribe_tick
q.subscribe_tick('2330', odd_lot=False)   # 逐筆成交

# 四種通道，各自 set_cb_X + subscribe_X：
#   tick    逐筆成交    subscribe_tick(symbol, odd_lot=False)
#   bidask  五檔買賣    subscribe_bidask(symbol, odd_lot=False)
#   all     成交+五檔   subscribe_all(symbol, odd_lot=False)
#   kbar    K 棒       subscribe_kbar(symbol, minute=1)   # minute ∈ 1/3/5/15/30/60

q.get_subscriptions()                 # 目前訂了哪些
q.unsubscribe(label)                  # 退訂單一；unsubscribe_all() 全退
```

**三者的差異**：`Quote`（台股）與 `USQuote`（美股）有 `odd_lot`（零股）與 `kbar`；
`FutQuote`（期貨）**沒有 `odd_lot`、沒有 `kbar`**，只有 tick / bidask / all，且 `symbol`
要帶月份（如 `TXFH6`）。多通道並行用 `api.Quote[n]`（`n` 是通道編號）。

**三個會踩的坑：**

- **`set_cb` 必須在 `subscribe` 之前**，而且是覆蓋式（不是累加）。
- **`SWMarketData` 的屬性永遠都在，但值不一定是這一筆的。** 先看 `data_type`：
  `2`=這筆成交（有 `close`/`volume`）、`3`=累計量（有 `total_volume`）、
  `1`=五檔更新（`close` 是殘值！）、`0`=快照、`4`=日高低、`5`=開盤。
  用 `d.summary()` 會只吐出這一筆真正帶的欄位。
- **回呼在背景執行緒**跑，改共享狀態要加 lock。

單位：整股張／零股股／期貨口。細節見 [swquote.md](https://snowyowlstrategy.github.io/swquote/) 三頁。

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

`rp.report`（27 項 KPI）、`rp.trade_detail`（逐筆交易）、`rp.trade_main`（委託）、
`rp.position`、`rp.daily`（權益曲線）、`rp.dashboard()`（plotly 報表）。

**沒有 `rp.trade`**——逐筆在 `rp.trade_detail`。欄位定義見 [backtest-report.md](https://snowyowlstrategy.github.io/backtest-report/)。

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
[backtest-data.md](https://snowyowlstrategy.github.io/backtest-data/)（餵資料與週期）、[backtest-report.md](https://snowyowlstrategy.github.io/backtest-report/)（27 項 KPI 定義）、
[backtest-tools.md](https://snowyowlstrategy.github.io/backtest-tools/)（指標與尋優）。

## 其他命名空間

| | 用途 | 文件 |
|---|---|---|
| `api.Data` / `api.USData` | 表格式查詢（清單、財報、分K），`get(table, *args)` | [data-get.md](https://snowyowlstrategy.github.io/data-get/) |
| `api.MSMP` / `api.USMSMP` | 矩陣式歷史資料 日期 × 股票代號 | [msmp.md](https://snowyowlstrategy.github.io/msmp/)、[msmp-tables.md](https://snowyowlstrategy.github.io/msmp-tables/) |
| `api.SWQuote` | StarWave 即時行情訂閱（**首選**，台美期統一；沒訂閱的帳戶不掛此屬性，先 `hasattr`） | [swquote.md](https://snowyowlstrategy.github.io/swquote/) |
| `api.Quote` / `api.USQuote` / `api.FutQuote` | 即時行情舊路徑（台股／美股／期貨分開）。**只在沒有 `api.SWQuote` 時才用** | [swquote.md](https://snowyowlstrategy.github.io/swquote/) |
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

