# Taiwan Stock Data Pipeline QA Pack (Free Sample)

台股資料管線的**靜默失敗**案例庫 + 機器可讀檢查規則：這些事故的共同點是
API 全部回傳 HTTP 200、沒有任何 exception —— 錯的不是程式，是資料的
**時間、覆蓋、日曆**，只有 invariant 檢查抓得到。本 repo 是免費樣本
（3 案例），讓串接 TWSE / TPEX / FinMind / 券商 API 的台股量化與
FinTech 團隊，不用親自賠一次錢才學到同一課。

## 免費樣本：3 個真實事故

| 案例 | 症狀一句話 | 類別 |
|---|---|---|
| [Case 01](case_01_intraday_stale_baseline.md) | API 回傳成功，但資料不是今天的 —— 盤中用 EOD 源昨收當即時 baseline，權證 strike 篩選區間整段位移 70-90 元 | 延遲與 stale |
| [Case 02](case_02_tpex_missing_universe.md) | 明明 App 看得到 339 檔，我的主檔說 0 檔 —— 只拉 TWSE 漏掉整個 TPEX 上櫃，全市場靜默缺 12,819 筆（23.7%） | 完整性與筆數 |
| [Case 03](case_03_non_trading_day_signals.md) | 颱風假那天策略照樣「產生了 47 筆訊號」—— 排程只認星期幾不認交易所日曆，加上週末排程三個月累積 100 筆假訊號 | 日曆與排程 |

每個案例固定八節：真實症狀 / 為什麼不會報錯 / 受影響結果（量化）/
最小重現樣本 / 檢查條件（invariant）/ 建議重試回補策略 / 驗收標準 /
已知例外。案例內的 Python 片段只用標準函式庫，在 repo 根目錄直接執行
即可重現。

## checks.yaml — 8 條機器可讀檢查規則

[`checks.yaml`](checks.yaml) 把三個案例的教訓收斂成 8 條**來源無關
invariant**（id / severity / 適用資料源 / pseudo-logic），可直接落地成
pandera、great_expectations 或自寫 assert。用法示例（以 Case 02 的
交易所覆蓋檢查為例）：

```python
import csv
import yaml  # pip install pyyaml

rules = {r["id"]: r for r in yaml.safe_load(open("checks.yaml", encoding="utf-8"))["checks"]}
rule = rules["universe_exchange_coverage"]

rows = list(csv.DictReader(open(rule["fixtures"]["fail"], encoding="utf-8")))
exchanges = {r["exchange"] for r in rows}
assert {"TSE", "OTC"} <= exchanges, (
    f"[{rule['severity']}] {rule['name']}: exchange 只有 {exchanges}，"
    f"詳見 {rule['case']}"
)
```

對 `fixtures.fail` 跑必須炸、對 `fixtures.pass` 跑必須過 —— 每條規則都
附這兩面的樣本資料，接進 CI 就是現成的資料品質回歸測試。

## fixtures/ — 全合成測試資料

`fixtures/` 內每個案例一組「壞資料長什麼樣 vs 好資料長什麼樣」的樣本。
**所有 fixtures 均為合成資料**：數字場景取自真實事故（例如 23.7% 覆蓋
缺口、strike 區間位移），但不含任何真實抓取的行情資料，可安心納入
測試套件散布。

## 完整版 — 20 案例

免費 3 案例只是「延遲 / 完整性 / 日曆」三類各抽一個。完整版覆蓋五大類
20 個案例（延遲與 stale / 完整性與筆數 / Schema 與型別 / 跨來源一致性 /
回補與自癒），每案例同樣附可跑重現樣本、checks.yaml 規則、合成 fixtures、
驗收標準與已知例外：

**→ [完整版 20 案例](https://tw-data-qa.pages.dev/?utm_source=github&utm_medium=readme)**

## 授權

- `checks.yaml`、`fixtures/`、README 內程式片段：**MIT**（見 [LICENSE](LICENSE)）——
  機器可讀規則與合成資料，歡迎直接放進你的 pipeline。
- 案例文件（`case_*.md`）：**[CC BY-NC 4.0](https://creativecommons.org/licenses/by-nc/4.0/deed.zh-hant)**
  （姓名標示—非商業性）。商業使用與完整 20 案例請見
  [官網](https://tw-data-qa.pages.dev/?utm_source=github&utm_medium=readme)。

## 邊界聲明

- 本包內容為固定交付物：案例文件、檢查規則、合成測試資料。
- 不含諮詢、debug 服務或更新承諾。
