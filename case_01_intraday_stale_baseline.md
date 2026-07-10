# Case 01 — API 回傳成功，但資料不是今天的：盤中用 EOD 資料源的昨收當即時 baseline

> 類別：延遲與 stale ｜ 嚴重度：Critical ｜ 對應檢查規則：`stale_baseline_date_mismatch`、`intraday_requires_realtime_source`

---

## 1. 真實症狀

盤中 10:05，我的權證篩選器跑完了，沒有任何錯誤。log 乾乾淨淨：HTTP 200、
回傳非空、欄位齊全、候選清單正常產出。

問題是標的 6274 當天盤中已經 +6.8%（$1540 → $1645），而篩選器用的
underlying baseline 是**昨天的收盤價 $1540** —— 因為資料源是一個
EOD（end-of-day）日線 API，盤中打它，它「成功地」回給你最近一個交易日的收盤。

篩出來的整份 ITM 70-85% 候選清單，strike 區間是 1078-1309。
用真正的即時價 $1645 算，應該是 1152-1398。**兩份清單幾乎不重疊。**

這個 bug 藏了多久沒人知道，因為對波動小的標的它「碰巧對」：
同一天 3711 昨收 $627 vs 即時 $619（-1.3%）、6770 昨收 $80.7 vs 即時
$81.10（+0.5%），結論差異小到看不出來。只有標的大漲大跌那天才炸，
而那天恰好是你最需要篩選器正確的一天。

## 2. 為什麼不會報錯

- EOD API 的合約本來就是「回傳最近一個已結算交易日的資料」。盤中呼叫它，
  回傳 200 + 非空 data 是**正確行為**，不是故障。
- 錯的是呼叫端的隱含假設：「API 成功 = 資料是現在的」。沒有任何一層會替你
  檢查 `data date == today`。
- 分鐘級資料同理：部分免費/中介源（如 FinMind 分 K）盤中有 **15 分鐘以上
  lag**，甚至當日資料是**盤後批次上架**（分 K 約 16:00 後、日 K 約 18:30 後
  才齊）。盤中打，拿到的是空檔或舊檔，一樣不報錯。
- 下游全部照常運作：篩選邏輯收到一個合法的 float，算出一份合法的清單。
  型別對、格式對、只有「時間」是錯的。

## 3. 受影響結果（量化）

| 項目 | 數字 |
|---|---|
| baseline 價差（6274，+6.8% 日） | $1540 vs $1645，差 $105 |
| ITM 70-85% strike 篩選區間位移 | **差 70-90 元**（1078-1309 vs 1152-1398） |
| 候選集重疊度 | 幾乎為零 —— 兩份完全不同的推薦清單 |
| 盤中動態可見度 | 0%（2303 當日盤中高點 $149 殺到 $138，-7% 的 swing，用昨收 $142 完全看不見） |

衍生損害：以錯誤 strike 區間下單的滑價／錯單、以昨收計算的槓桿與 delta
全部失真、回測若混入這種 baseline 會產生無法重現的假 edge。

## 4. 最小重現樣本

fixtures：`fixtures/case_01_eod_api_response.json`（壞：盤中打 EOD API 的實際回傳形狀）
vs `fixtures/case_01_realtime_snapshot.json`（好：即時 snapshot 含 timestamp）。
均為合成資料，數字取自真實事故。

```python
import json
from datetime import date

TODAY = date(2026, 5, 29)  # 模擬「現在是盤中 10:06」

# --- 壞路徑：EOD 日線 API,盤中呼叫 ---
with open("fixtures/case_01_eod_api_response.json", encoding="utf-8") as f:
    eod = json.load(f)

last_row = eod["data"][-1]
print(f"HTTP {eod['status']}, rows={len(eod['data'])}  <- 一切看似成功")
print(f"latest date={last_row['date']}  close={last_row['close']}")

def itm_strike_range(px, lo=0.70, hi=0.85):
    return round(px * lo), round(px * hi)

stale_range = itm_strike_range(last_row["close"])   # (1078, 1309)

# --- 好路徑：即時 snapshot,帶 timestamp ---
with open("fixtures/case_01_realtime_snapshot.json", encoding="utf-8") as f:
    snap = json.load(f)

live_range = itm_strike_range(snap["last"])          # (1152, 1398)
print(f"stale strike range = {stale_range}")
print(f"live  strike range = {live_range}")

# --- invariant 檢查:這行就是整個案例的教訓 ---
assert last_row["date"] == TODAY.isoformat(), (
    f"STALE_BASELINE: API 回傳成功,但最新資料日期是 {last_row['date']},"
    f" 不是 {TODAY}。盤中禁止以此當即時 baseline。"
)
```

執行結果：兩個 strike range 相差 70-90 元，`assert` 以 `STALE_BASELINE` 炸出 ——
這正是生產環境該發生但沒發生的事。

## 5. 檢查條件（invariant）

**I-1「日期同日」**：任何被當作「即時 baseline」使用的價格，其資料日期
（`date` 欄或 timestamp 的日期部分）**必須等於今天的交易日**。
→ `checks.yaml: stale_baseline_date_mismatch`

**I-2「新鮮度上限」**：盤中（09:00-13:30）使用的報價，其 timestamp 距現在
不得超過 N 分鐘（建議 N=5；日內策略建議 N=1）。沒有 timestamp 欄位的資料源，
一律視為不合格的盤中即時源。
→ `checks.yaml: intraday_requires_realtime_source`

**I-3「來源分工」**：每個資料源標記 `eod` 或 `realtime` 角色。`eod` 源
（FinMind 日線、TWSE OpenAPI、Yahoo Finance）在盤中路徑出現 = 直接 fail，
不看資料內容。台股盤中即時 last/bid/ask 應走券商 API（如 Shioaji
`api.snapshots()`）。
→ `checks.yaml: intraday_requires_realtime_source`（`source_role` 欄）

## 6. 建議重試/回補策略

- **不要 retry**。這不是暫時性故障 —— EOD 源 retry 一百次還是昨收。
  正確處置是**切源**：盤中路徑改走即時 snapshot API。
- **降級策略**：即時源不可用（斷線/未登入）時，寧可讓篩選器**明確停止並告警**，
  也不要 fallback 到 EOD 昨收繼續產出。stale 產出比沒有產出更貴。
- **回補**：EOD 源的正確用途是盤後。當日日 K 依源不同在 16:00-18:30 後才穩定
  上架，回補排程放 18:30 之後，且回補後跑一次 I-1 驗證資料日期真的是當天。
- **法人/籌碼類**（T86 等）另有公布時序：TWSE 官方（rwd 介面）盤後
  15:00-17:00 入庫、OpenAPI 慢 1 天、FinMind 再慢 1-2 天。拉不到先降層
  往官方源試，不要假設「還沒公布」。

## 7. 驗收標準

1. 對 fixtures 跑檢查：`case_01_eod_api_response.json` 在「盤中 + 即時用途」
   情境下必須觸發 `stale_baseline_date_mismatch`（severity: critical）。
2. `case_01_realtime_snapshot.json` 在同情境下必須通過（timestamp 為今日盤中）。
3. 生產 pipeline 中每一條「盤中讀價」的 code path，都能指出它掛的是哪條
   invariant 檢查；找不到對應檢查的 path 視為未覆蓋。
4. 人工演練：把系統時鐘視為交易日 T 的 10:00，餵 T-1 的日線資料，pipeline
   必須拒絕產出而非靜默給出候選清單。

## 8. 已知例外

- **盤後與回測路徑**：收盤後（13:30 之後）拿當日或歷史日線做分析、對齊昨日
  EOD 基準算 paper P&L，用 EOD 源完全正當，I-1 不適用（此時「今天的交易日」
  定義為最近一個已收盤交易日）。
- **開盤前**（08:00-09:00）：市場尚無今日成交，baseline 本來就是昨收，
  date == T-1 是預期行為。檢查應以「當前市場狀態」為條件，而非一律要求今日。
- **T-1 為假日**：週一盤前拿到上週五資料屬正常，「今天的交易日」需用交易日曆
  換算，不能用 `date.today() - 1`（見 Case 03）。
- **低波動標的碰巧對**：檢查不能用「結果差多少」判定，必須檢查日期本身 ——
  價差小不代表流程對，只代表這次運氣好。
