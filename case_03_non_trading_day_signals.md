# Case 03 — 颱風假那天，我的策略照樣「產生了 47 筆訊號」：非交易日排程照跑產出假訊號

> 類別：回補與自癒 ｜ 嚴重度：High ｜ 對應檢查規則：`signal_date_is_trading_day`、`calendar_confirmed_absence`、`scheduler_trading_day_gate`

---

## 1. 真實症狀

7/10 台股因颱風全天休市。我的每日訊號排程不知道 —— 它只知道「今天是週五」，
照表操課：拉資料、跑模型、輸出 **47 筆訊號**，寫進追蹤資料庫，exit code 0，
一切「成功」。

那 47 筆訊號的依據是什麼？休市日根本沒有新行情，上游 API 回的是**最近一個
交易日的舊資料**（見 Case 01 的 stale 機制）——所以模型是拿 7/9 的行情，
產出標記為 7/10 的訊號。日期是假的，訊號自然全是假的。

清理時順藤摸瓜，發現更久的洞：另一條排程建立時**忘了限定週一到週五**，
從 4 月起每個週六日照跑，累積了 **100 筆週末假訊號**，安安靜靜躺在績效
統計裡三個月。兩個洞同一個根因：**排程器的日曆不是交易所的日曆**。

## 2. 為什麼不會報錯

- 排程器（Windows schtasks / cron）只認星期幾，不認「台股今天有沒有開盤」。
  颱風臨時休市、選舉日、彈性放假、年節連假的調整上班日 —— 沒有一個是
  「MON-FRI」規則能表達的。
- 休市日打行情 API 通常**不會回錯誤**：EOD 源回最近交易日資料（HTTP 200、
  非空、schema 正確），部分即時源回最後一筆快照。每一層都「成功」。
- 模型收到合法形狀的資料就產出合法形狀的訊號。整條 pipeline 沒有任何一步
  需要知道「今天市場有沒有開」——除非你明確教它。
- 週末假訊號更隱蔽：沒人在週六看報表，假訊號直接落庫，等到月度績效統計
  才以「樣本數怎麼比交易日多」的形式露餡 —— 如果有人剛好去數的話。

## 3. 受影響結果（量化）

| 項目 | 數字 |
|---|---|
| 單一颱風休市日產出的假訊號 | **47 筆**（一天） |
| 週末排程未限定平日的累積假訊號 | **100 筆**（約 3 個月無人發現） |
| 合計污染追蹤資料庫 | **147 筆**需回頭 purge |
| 績效統計影響 | 假訊號以 stale 價結算，稀釋/扭曲勝率與平均報酬；本例清洗後主策略數字由 n=400 修正為 n=390 |

衍生損害：假訊號若接自動下單就是真金白銀的錯單；接推播就是假警報消耗
使用者信任；混進訓練集就是用不存在的交易日污染模型。

## 4. 最小重現樣本

fixtures：`fixtures/case_03_signals_bad.csv`（壞：含颱風日 7/10 與週六訊號）、
`fixtures/case_03_signals_good.csv`（好：全部落在交易日）、
`fixtures/case_03_trading_calendar.csv`（合成交易日曆，含 7/10 颱風休市標記）。
均為合成資料，筆數場景對應真實事故。

```python
import csv
from datetime import date

def load_calendar(path):
    with open(path, newline="", encoding="utf-8") as f:
        return {r["date"]: r for r in csv.DictReader(f)}

def load_signals(path):
    with open(path, newline="", encoding="utf-8") as f:
        return list(csv.DictReader(f))

cal = load_calendar("fixtures/case_03_trading_calendar.csv")

for path in ("fixtures/case_03_signals_bad.csv",
             "fixtures/case_03_signals_good.csv"):
    signals = load_signals(path)
    bad = []
    for s in signals:
        d = s["signal_date"]
        entry = cal.get(d)
        if entry is None:
            # 日曆沒這天 != 非交易日,可能只是日曆沒更新 -> 另行告警,不誤殺
            print(f"  WARN CALENDAR_GAP: {d} 不在交易日曆內,需先補日曆")
            continue
        if entry["is_trading_day"] != "1":
            bad.append((s["signal_id"], d, entry["note"]))
    print(f"{path}: {len(signals)} 筆訊號, 非交易日假訊號 {len(bad)} 筆")
    for sig_id, d, note in bad:
        print(f"  FAKE_SIGNAL: {sig_id} @ {d} ({note or 'non-trading day'})")
    assert not bad, "NON_TRADING_DAY_SIGNAL: 非交易日不得產生訊號"
```

執行結果：`bad` 檔抓出 7/10（颱風休市）與週六的假訊號並炸出
`NON_TRADING_DAY_SIGNAL`；`good` 檔全過。注意 `CALENDAR_GAP` 分支 ——
「日曆查無此日」與「確認非交易日」是兩件事，混為一談會誤殺好資料（見第 8 節）。

## 5. 檢查條件（invariant）

**I-1「訊號日必為交易日」**：任何帶日期的產出（訊號、榜單、日報），其日期
必須存在於交易日曆且 `is_trading_day == true`。在**寫入端**（ingest）擋，
不只在排程端擋 —— 排程端擋板擋不住手動補跑與時鐘錯亂。
→ `checks.yaml: signal_date_is_trading_day`

**I-2「證實缺席才擋」**：判定「今天休市」必須以**正面證據**為準（官方日曆
標記、或行情資料經探測確認當日全市場無成交），不能以「拉不到資料」推論
休市。資料未到 ≠ 市場沒開（見 Case 01 的公布時序）。
→ `checks.yaml: calendar_confirmed_absence`

**I-3「排程雙保險」**：排程規則本身限定平日（MON-FRI）只是第一層；第二層
是 job 開頭查交易日曆，非交易日直接 exit 並記一筆「今日休市，跳過」的
明確 log —— 讓「沒跑」可稽核，與「掛了沒跑」區分開。
→ `checks.yaml: scheduler_trading_day_gate`

## 6. 建議重試/回補策略

- **交易日曆是基礎設施**：台股臨時休市（颱風/選舉）無法離線預知，日曆需有
  資料源（TWSE 官網公告/行情探測）定期刷新。日曆本身也要監控 —— 一份三個月
  沒更新的日曆比沒有日曆更危險。
- **回頭 purge**：擋板上線只解決 forward，歷史庫裡的假訊號要主動清：
  以交易日曆 join 全表，date 不在交易日的整批標記/刪除，並重算所有受污染的
  績效統計（本例 purge 147 筆後重驗結論）。
- **休市日的正確行為**：跳過並留痕（log「2026-07-10 market closed
  (typhoon), skipped」），不要靜默 exit 0 —— 下游監控需要能區分
  「今天正常休市」與「今天排程掛了」。
- **手動補跑防呆**：backfill 腳本同樣要過 I-1。歷史回補最容易繞過排程端
  擋板，寫入端 invariant 是最後防線。

## 7. 驗收標準

1. 對 fixtures 跑檢查：`case_03_signals_bad.csv` 必須抓出全部非交易日訊號
   （颱風日 + 週末），`case_03_signals_good.csv` 零誤報。
2. 演練「臨時休市」：把日曆中任一平日標記為休市後觸發排程，pipeline 必須
   跳過產出並留下明確 skip log，且追蹤資料庫零新增。
3. 演練「資料未到」：交易日但行情 API 暫時回空，pipeline **不得**判定休市
   （I-2），應走重試/延後路徑並告警。
4. 歷史庫全表掃描：`signal_date` 不在交易日曆的筆數 = 0。

## 8. 已知例外

- **「資料未到」vs「非交易日」必須分流**：真實踩過的反例 —— 官方源某交易日
  延遲公布，若用「拉不到 = 休市」邏輯，會把**真交易日誤判成假日**造成資料洞。
  I-2 的「正面證據」要求就是為此而設；近幾日資料缺席只告警不擋。
- **半日市/提早收盤**：台股偶有非全日交易情境（如除夕前）。`is_trading_day`
  可能需要第三種狀態，全日/半日對日內策略是不同語意。
- **跨市場系統**：台股休市不代表美股休市。日曆必須 per-market，用錯市場的
  日曆與沒有日曆同樣危險。
- **休市日的合法產出**：某些產出在非交易日是正當的（週報、以 T-1 資料做的
  盤後分析）。I-1 管的是「宣稱屬於某交易日的訊號/行情類產出」，報表類產出
  應以「資料基準日」而非「產出日」受檢。
