# 策略規格書：Dan Zanger 帶量突破動量策略 (Volume Breakout Strategy)

## 1. 策略概述 (Strategy Overview)
- **策略名稱**：Dan Zanger 權值股帶量突破策略 (DZ Volume Breakout)
- **策略屬性**：趨勢順勢交易 / 動量進攻型 (Momentum Trend-Following)
- **核心哲學**：尋找低波動度、窄幅箱型整理後的爆發起漲點，要求成交量放大以確認真實動能，且嚴格控制進場追價容忍度，降低假突破風險。
- **目標標的池**：台灣集中市場市值前 100 大權值股 (Top 100 Weighted Stocks)
- **資料週期**：日線 (Daily / 還原日線 Adjusted Daily)，盤後運算選股。

---

## 2. 策略參數 (Input Parameters)

| 參數名稱 | 變數代碼 | 預設值 | 說明 |
| :--- | :--- | :--- | :--- |
| 整理觀察天數 | `BaseDays` | 30 | 定義箱型整理與阻力/支撐的 Lookback 週期 |
| 整理區間最大幅度% | `MaxBasePct` | 20.0 | 箱型上下邊界收斂上限，避免高波動度整理 |
| 基準均量天數 | `VolDays` | 20 | 計算突破前成交量均值的週期 |
| 突破最低量比 | `MinVolRatio` | 1.5 | 突破當日成交量相對過去均量的最低倍數 ($V_t \ge 1.5 \times \text{MA}_{20}(V)$) |
| 距突破價最大漲幅% | `MaxChasePct` | 5.0 | 突破點上方最大允許追價邊界，超過不進場 |

---

## 3. 邏輯定義與演算法 (Logic & Algorithm)

### 3.1 關鍵指標計算 (排除當日 $t=0$)
- **箱型阻力價 (Pivot Price)**：
  $$PivotPrice = \max(High_{t-1}, High_{t-2}, \dots, High_{t-BaseDays})$$
- **箱型支撐價 (Base Low)**：
  $$BaseLow = \min(Low_{t-1}, Low_{t-2}, \dots, Low_{t-BaseDays})$$
- **基準成交均量 (Prior Avg Volume)**：
  $$PriorAvgVol = \frac{1}{VolDays} \sum_{i=1}^{VolDays} Volume_{t-i}$$

### 3.2 進場條件 (Entry Conditions)
當日收盤滿足以​​下所有條件時，觸發買進訊號 (`Ret = 1`)：
1. **箱型收斂**：整理區間幅度 $\le$ 最大允許幅度
   $$\frac{PivotPrice}{BaseLow} - 1 \le \frac{MaxBasePct}{100}$$
2. **實質收盤突破**：
   $$Close > PivotPrice$$
3. **動能量能激增**：
   $$Volume \ge PriorAvgVol \times MinVolRatio$$
4. **追價風控上限**：收盤價未過度偏離 Pivot
   $$Close \le PivotPrice \times \left(1 + \frac{MaxChasePct}{100}\right)$$

### 3.3 出場機制 (Exit Conditions)
- **主要停利/趨勢保護出場**：
  - **條件**：收盤價跌破「投信成本線」（投信持股成本均線）。
  - **原理**：作為大型權值股主力機構籌碼的動能退場信號，保護波段利潤。
- **回測警示與無效設定 (Anti-Patterns)**：
  - 避免採用固定 $8\%$ 停損停利機制跑全市場（全市場雜訊過多、假訊號密集，易大幅侵蝕期望值）。

---

## 4. 實證回測特徵與觀察 (Backtest Insights)

- **回測期間**：12 年歷史數據。
- **標的分群敏感度**：
  - **前 100 大權值股**：勝率超過 80%，表現為高勝率攻擊型策略。
  - **全體普通股 (Broad Market)**：假突破比率顯著上升，雜訊過多導致績效大幅衰退。
- **風險特徵**：
  - 由於追隨強勢波段行情，策略 MDD (最大回撤) 相對較高。
  - 適合定位於大盤多頭進攻或總體趨勢偏多時配置。

---

## 5. XScript 選股源代碼 (XS Implementation)

```pascal
{@type:filter}
// 區間整理 + 帶量突破 + 不追高
// 台股，日線或還原日線，盤後執行。

Input:
    BaseDays(30, "整理觀察天數"),
    MaxBasePct(20, "整理區間最大幅度%"),
    VolDays(20, "基準均量天數"),
    MinVolRatio(1.5, "突破最低量比"),
    MaxChasePct(5, "距突破價最大漲幅%");

Var:
    PivotPrice(0),
    BaseLow(0),
    PriorAvgVol(0),
    BaseWidthPct(0),
    BreakoutPct(0),
    BreakoutVolRatio(0),
    FiveDayPct(0);

SetBarFreq("D", "AD");
SetTotalBar(3);
SetBackBar(MaxList(BaseDays, VolDays, 5) + 2);

if BaseDays < 2 or VolDays < 1 then
    RaiseRunTimeError("整理天數須至少2天，均量天數須至少1天");

if MaxBasePct <= 0 or MinVolRatio <= 1 or MaxChasePct <= 0 then
    RaiseRunTimeError("區間與追價百分比須大於0，量比須大於1");

// 基準計算排除當日 (Offset = 1)
PivotPrice = Highest(High[1], BaseDays);
BaseLow = Lowest(Low[1], BaseDays);
PriorAvgVol = Average(Volume[1], VolDays);

Ret = 0;
BaseWidthPct = 0;
BreakoutPct = 0;
BreakoutVolRatio = 0;
FiveDayPct = 0;

if GetFieldStartOffset("收盤價") >= MaxList(BaseDays, VolDays, 5)
   and PivotPrice > 0 and BaseLow > 0
   and PriorAvgVol > 0 and Close[5] > 0 then
begin
    BaseWidthPct = (PivotPrice / BaseLow - 1) * 100;
    BreakoutPct = (Close / PivotPrice - 1) * 100;
    BreakoutVolRatio = Volume / PriorAvgVol;
    FiveDayPct = (Close / Close[5] - 1) * 100;

    // 訊號判定
    if PivotPrice <= BaseLow * (1 + MaxBasePct / 100)
       and Close > PivotPrice
       and Volume >= PriorAvgVol * MinVolRatio
       and Close <= PivotPrice * (1 + MaxChasePct / 100) then
        Ret = 1;
end;

OutputField(1, PivotPrice, 2, "突破價");
OutputField(2, BaseWidthPct, 2, "整理幅度%");
OutputField(3, BreakoutVolRatio, 2, "突破量比");
OutputField(4, BreakoutPct, 2, "距突破價%");
OutputField(5, FiveDayPct, 2, "近5日漲幅%");
