# Case 02 — 明明 App 看得到，我的資料庫卻說「0 檔」：商品主檔只拉 TWSE 漏掉 TPEX 上櫃

> 類別：完整性與筆數 ｜ 嚴重度：Critical ｜ 對應檢查規則：`universe_exchange_coverage`、`known_otc_probe_nonzero`、`universe_count_floor`

---

## 1. 真實症狀

查詢標的 6274 的認購權證，我的主檔（master cache）回答：**0 檔**。
程式沒有錯、cache 沒有壞、41,201 筆權證資料完整無缺。

打開券商 App 一看：6274 掛著 **339 檔** Call。

再查兩檔：2337 主檔 0 檔（實際 193 檔）、2492 主檔 0 檔（實際 177 檔）。
不是資料髒，是**整個上櫃（TPEX/OTC）市場從來沒進過主檔**。主檔建置時只打了
TWSE OpenAPI 的上市權證清單 `t187ap37_L`，而上櫃標的對應的權證是上櫃權證，
登記在櫃買中心（TPEX）自己的 OpenAPI `mopsfin_t187ap37_O` —— 一個完全獨立的
機構、完全獨立的 API base URL。

最陰的是「0 檔」這個答案**語意上完全合法**：很多標的本來就真的沒有權證。
下游看到 0 不會懷疑，只會跳過。這個洞開著跑了數週，直到有人拿 App 對過來。

## 2. 為什麼不會報錯

- TWSE OpenAPI 的回應是**正確且完整的** —— 對「上市權證」而言。它的合約
  就只覆蓋上市（TSE），漏上櫃不是它的 bug，是呼叫端的覆蓋假設錯了。
- 台灣市場結構：上市歸 TWSE（證交所）、上櫃/興櫃歸 TPEX（櫃買中心），
  **兩個機構、兩套 OpenAPI，沒有任何一個端點回「全市場」**。任何只打單一
  機構的「全市場」主檔，天生缺一角。
- 41,201 筆是個「看起來很大」的數字，過任何 `len(data) > 0` 或
  `len(data) > 1000` 的健全性檢查都輕鬆。
- 查無資料回 0 筆與空清單，是查詢層的正常語意，不拋例外。缺的是「這個 0
  是真 0 還是覆蓋缺口」的判斷 —— 程式自己判斷不了，必須靠外部 invariant。

## 3. 受影響結果（量化）

| 項目 | 數字 |
|---|---|
| 缺漏筆數 | **12,819 筆上櫃權證**（修復後全檔 54,020 筆 = 上市 41,201 + 上櫃 12,819） |
| 缺漏比例 | **23.7%** 的全市場權證不存在於主檔 |
| 個別標的假象 | 6274：0 vs 339 ｜ 2337：0 vs 193 ｜ 2492：0 vs 177 |
| 受害標的類型 | 系統性偏差 —— 上櫃多為中型/題材股，恰是權證交易最活躍的族群之一 |

衍生損害：篩選器對整個上櫃族群永遠回空 → 推薦清單系統性偏向大型上市股；
回測 universe 缺 23.7% → 統計結論帶結構性偏差且無法靠加大樣本修復。

## 4. 最小重現樣本

fixtures：`fixtures/case_02_universe_twse_only.csv`（壞：只有 TSE 來源）
vs `fixtures/case_02_universe_merged.csv`（好：TWSE+TPEX 合併、含 `exchange` 欄）。
均為合成資料（代號結構仿真：上市權證多為 0 開頭、上櫃權證多為 7 開頭），
筆數比例對應真實事故的 41,201 / 12,819。

```python
import csv
from collections import Counter

# 已知「一定有上櫃權證」的探針標的(維護一份小清單即可)
OTC_PROBES = {"6274", "2337", "2492"}

def load_universe(path):
    with open(path, newline="", encoding="utf-8") as f:
        return list(csv.DictReader(f))

for path in ("fixtures/case_02_universe_twse_only.csv",
             "fixtures/case_02_universe_merged.csv"):
    rows = load_universe(path)
    by_exchange = Counter(r["exchange"] for r in rows)
    probe_hits = {c: sum(1 for r in rows if r["underlying_code"] == c)
                  for c in OTC_PROBES}
    print(f"\n{path}")
    print(f"  總筆數={len(rows)}  exchange 分佈={dict(by_exchange)}")
    print(f"  探針標的權證數={probe_hits}")

    # invariant 1: 全市場主檔必須同時含 TSE 與 OTC
    assert {"TSE", "OTC"} <= set(by_exchange), \
        "COVERAGE_GAP: 主檔缺少整個交易所來源(只拉了 TWSE 沒拉 TPEX?)"
    # invariant 2: 已知有權證的上櫃標的,查詢結果不得為 0
    assert all(v > 0 for v in probe_hits.values()), \
        f"OTC_PROBE_ZERO: 探針標的在主檔查無權證 {probe_hits}"
```

執行結果：`twse_only` 檔在 invariant 1 就炸出 `COVERAGE_GAP`；
`merged` 檔兩條全過。

## 5. 檢查條件（invariant）

**I-1「交易所覆蓋」**：宣稱「全市場」的主檔/universe，`exchange`（或等價
來源標記）欄的 distinct 值必須同時包含 TSE 與 OTC。建檔時就寫入來源標記，
沒有來源標記的主檔視為不可稽核。
→ `checks.yaml: universe_exchange_coverage`

**I-2「探針非零」**：維護一份小型探針清單（3-5 檔確定有上櫃權證/上櫃掛牌的
標的），每次主檔重建後查詢，任何一檔回 0 即 fail。這是抓「合法的 0」最便宜
的手段。
→ `checks.yaml: known_otc_probe_nonzero`

**I-3「筆數下限」**：主檔總筆數對照歷史基準（如 30 日中位數），單日掉幅
超過 10% 即告警 —— 23.7% 的缺口若有歷史基準比對，第一天就會被抓到。
→ `checks.yaml: universe_count_floor`

## 6. 建議重試/回補策略

- **不要 retry**。單源回應本身是完整的，retry 不會長出另一個交易所的資料。
  正確處置是**補源**：TWSE `t187ap37_L`（上市）+ TPEX `mopsfin_t187ap37_O`
  （上櫃）兩個端點都拉、合併寫入，並加 `exchange` 欄標記來源。
- **部分失敗處理**：兩源其中一個失敗時，不要默默用剩下那個當「全市場」。
  要嘛整批標記為 partial 並告警，要嘛沿用上一版完整主檔並標記 stale ——
  兩者都比「靜默半套」好。
- **回補**：修復後重跑歷史依賴（回測 universe、統計基準）；混過半套主檔的
  歷史結論一律重算，不能只修 forward。
- **通用化**：這條 invariant 適用於所有台股商品類別 —— 股票、權證、ETF、
  可轉債。凡是只打 TWSE 的「全市場」清單，預設就漏上櫃/興櫃。

## 7. 驗收標準

1. 對 fixtures 跑檢查：`case_02_universe_twse_only.csv` 必須觸發
   `universe_exchange_coverage`（severity: critical）；`case_02_universe_merged.csv`
   三條規則全過。
2. 探針清單查詢：6274/2337/2492（或自選的上櫃探針）在重建後的主檔中
   權證數 > 0。
3. 主檔 schema 含 `exchange`（或 `source`）欄位，且每筆非空。
4. 兩源合併後總筆數落在歷史基準 ±10% 內；首次建置以官方公布檔數交叉核對
   （量級參考：上市約 4 萬檔、上櫃約 1.3 萬檔，合計 5 萬檔級）。

## 8. 已知例外

- **刻意只做上市的系統**：若產品範圍明確定義為「僅上市」，I-1 不適用 ——
  但必須在主檔 metadata 明寫 `scope: TSE_only`，讓下游可稽核，而不是隱含假設。
- **探針標的自身變動**：探針標的可能因下市、轉上市（上櫃轉上市）、權證全數
  到期而合法歸零。探針清單需在 fail 時先人工確認一次再更新清單，不可自動移除。
- **興櫃/創櫃**：TPEX OpenAPI 覆蓋上櫃/興櫃/創櫃，但多數權證/衍生品分析
  不含興櫃。範圍要明確定義，「TWSE+TPEX 上櫃」與「含興櫃」是不同的 universe。
- **筆數基準的市場性波動**：權證每日有新發行與到期，總筆數自然漲跌；
  I-3 的 10% 門檻抓的是結構性缺口，1-3% 的日常波動屬正常。
