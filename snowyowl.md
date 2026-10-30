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
  表格查詢：api.Data／api.USData、get(table)、get_table(type=dic)、清單／分K／逐筆明細、
  v2 新增表（日_K線資料表、季_財務比率…）。
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

# snowyowl

台股／美股資料、積木選股與回測的 Python 套件。這份 skill 只講「必須遵守的規則」與「去哪裡查」；
API 細節、寫法範例都在文件裡，以文件為準。

完整文件在 <https://snowyowlstrategy.github.io/>，索引見 <https://snowyowlstrategy.github.io/llms.txt>，全文串接見 <https://snowyowlstrategy.github.io/llms-full.txt>。本檔可從 <https://snowyowlstrategy.github.io/snowyowl.md> 取得最新版。

## 必須遵守

1. **回測一律用 `api.bt`**：`api.bt.Core` 寫策略、`api.bt.generate_report` 出報表。不要自己用 pandas 算報酬。
2. **產出 `rp` 就收手**：程式結尾只有

   ```python
   print(rp.report)                         # 多檔組合用 rp.report[['Strategy', 'benchmark']]，整張會被截斷
   rp.dashboard(path='<策略名>_dashboard.html', view='web')   # 最常用：存檔並用瀏覽器開啟
   ```

3. **程式盡量乾淨**：只寫需求要的計算。不要另外印缺值比例、因子分布、名單、檔數、毛損益、曝險率這類東西，除非使用者自己要加工。
4. **回覆時介紹 `rp` 底下有什麼**（細節見 [backtest-report.md](https://snowyowlstrategy.github.io/backtest-report/)），讓使用者知道還能看什麼：

   | 成員 | 內容 |
   |---|---|
   | `rp.dashboard()` | 互動儀表板（最常用），四個分頁：總覽、績效分析、交易明細、個股分析（K 線與進出場） |
   | `rp.report` | 28 項績效 KPI（有 benchmark 就並列） |
   | `rp.trade_detail` | 逐筆交易：進出場時間與價格、損益、成本 |
   | `rp.trade_main` | 每一筆委託，不論成交與否 |
   | `rp.position` | 每日持倉部位 |
   | `rp.daily` | 每日權益與回撤 |
   | `rp.daily_close` | 每日收盤價 |
   | `rp.save()` | 全部存成一個 xlsx |
   | `rp.set_benchmark(data)` | 事後設定或更新對照基準 |

## 起手式

```python
import snowyowl as so
api = so.login('帳號', '密碼', type='account')   # 或 so.login(type='google')
```

登入細節見 [login.md](https://snowyowlstrategy.github.io/login/)；`api` 底下有哪些物件見 [api-structure.md](https://snowyowlstrategy.github.io/api-structure/)。

## 流程

照 [agent-playbook.md](https://snowyowlstrategy.github.io/agent-playbook/)：

1. **分流**：查事實 → `api.MSMP` 一行算；組條件選股 → 積木 `api.Block`；驗規則 → 回測 `api.bt`（[getting-started.md](https://snowyowlstrategy.github.io/getting-started/)）
2. **找寫法**：到 [recipes.md](https://snowyowlstrategy.github.io/recipes/) 找用得上的寫法，照使用者的規格組合，不要整份照抄
3. **規格逐條對應**：每一條規格先找積木（[block-reference-tw.md](https://snowyowlstrategy.github.io/block-reference-tw/)、[block-auto.md](https://snowyowlstrategy.github.io/block-auto/)）；
   做不到才手寫，並在程式註解寫「積木做不到：原因」
4. **執行**：寫完要跑，報錯就修。不能執行時要明說「未執行」
5. **交付**：`print(rp.report)`、打開儀表板，回覆時介紹 `rp` 底下有什麼（見上方「必須遵守」）

## 寫法規則

與 [agent-playbook.md](https://snowyowlstrategy.github.io/agent-playbook/) 的「寫法鐵則」相同：

| 規則 | 錯了會怎樣 |
|---|---|
| 欄名照 [msmp-tables.md](https://snowyowlstrategy.github.io/msmp-tables/) **一字不差** | `AttributeError` |
| 月／季頻一律 `index_str_to_date()` 轉公告日 | 用到未公告的資料（未來函數）、混進下市股 |
| 對齊用 `union → ffill → reindex`，不要直接 `reindex` | 公告日對不上換股日，整片 NaN |
| **先套可交易遮罩再排名** | 排名母體含買不到的股票 |
| 反向因子 `1 - p`，缺值填 `0.5` | 用負號會讓排名方向錯；缺值填 0 會懲罰沒資料的股票 |
| 價格門檻、成交額用**未還原** `Close`；`Volume` 是張要 `×1000` | 門檻差好幾倍、成交額差 1000 倍 |
| 日 K 從 `api.MSMP` 取，不用 `bt.Tools.get_stock_ohlc` | 頻率跟積木 `sig` 對不上 |
| 輪動用 `qty = 1/N`（百分比路徑）；小資金 `unit=1`（零股） | 固定口數時 `capital` 不參與計算；整股買不起被夾成 0 |
| 台股只填 `tax_short=0.003`（ETF `0.001`），`tax_long=0` | 證交稅收兩次 |
| 指標寫進 `self.char`（主圖）／`self.char1`（副圖） | 就會畫在 `show_monitor()` 與 `rp.dashboard()` 的 K 線 |

## 去哪裡查

| 要做什麼 | 文件 |
|---|---|
| 回測寫法範例（單檔、積木訊號、多檔、期貨、輪動、IC、尋優、多因子） | [recipes.md](https://snowyowlstrategy.github.io/recipes/) |
| 積木怎麼組、怎麼跑 | [block.md](https://snowyowlstrategy.github.io/block/)、[block-con.md](https://snowyowlstrategy.github.io/block-con/)、[block-screener.md](https://snowyowlstrategy.github.io/block-screener/)、[block-pool.md](https://snowyowlstrategy.github.io/block-pool/) |
| 原生積木做不到的條件、白話 → 參數對照 | [block-auto.md](https://snowyowlstrategy.github.io/block-auto/) |
| 某個積木的參數與合法值 | [block-reference-tw.md](https://snowyowlstrategy.github.io/block-reference-tw/)、[block-reference-us.md](https://snowyowlstrategy.github.io/block-reference-us/) |
| 積木的已知落差 | [block-caveats.md](https://snowyowlstrategy.github.io/block-caveats/) |
| MSMP 有哪些表與欄位、常用查詢 | [msmp.md](https://snowyowlstrategy.github.io/msmp/)、[msmp-tables.md](https://snowyowlstrategy.github.io/msmp-tables/) |
| 表格式查詢（`api.Data`）有哪些表（`get_table(type='dic')` 可用程式列舉） | [data-get.md](https://snowyowlstrategy.github.io/data-get/)、[data-get-table.md](https://snowyowlstrategy.github.io/data-get-table/) |
| `get_table()`「新增」段的 28 張（`日_`／`週_`／`月_`／`季_`／`事件_` 開頭） | [data-v2-tables.md](https://snowyowlstrategy.github.io/data-v2-tables/) |
| 各頻率的歷史長度、期貨資料在哪 | [data-history.md](https://snowyowlstrategy.github.io/data-history/) |
| 盤中、分K、逐筆明細 | [intraday.md](https://snowyowlstrategy.github.io/intraday/) |
| 美股 | [usmsmp.md](https://snowyowlstrategy.github.io/usmsmp/)、[usdata.md](https://snowyowlstrategy.github.io/usdata/) |
| 回測框架 | [backtest.md](https://snowyowlstrategy.github.io/backtest/)、[backtest-strategy.md](https://snowyowlstrategy.github.io/backtest-strategy/)、[backtest-order.md](https://snowyowlstrategy.github.io/backtest-order/)、[backtest-data.md](https://snowyowlstrategy.github.io/backtest-data/)、[backtest-report.md](https://snowyowlstrategy.github.io/backtest-report/)、[backtest-tools.md](https://snowyowlstrategy.github.io/backtest-tools/) |
| 錯誤訊息、不報錯但結果是錯的 | [errors.md](https://snowyowlstrategy.github.io/errors/) |
