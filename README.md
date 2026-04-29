# 📘 投資型保單 GMDB 模擬與解約行為分析

## Unit-Linked Insurance Simulation with GMDB and Dynamic Lapse Behavior

### 📌 專案簡介

本專案建立投資型保單（Unit-Linked Insurance）的 Monte Carlo Simulation 模型，參考實際商品設計，分析保戶解約行為（lapse behavior）對死亡給付、死亡成本與公司獲利的影響。

模型結合三個重要元素:

1. 使用 GBM 模擬投資標的報酬路徑，透過 Monte Carlo 生成帳戶價值（AV）的分布。
2. 參考實際商品保單結構、死亡率與 COI 費率，計算 Death Benefit 與 Death Cost 。
3. 引入動態解約模型（dynamic lapse model），使保戶行為隨帳戶價值、保證水準與市場環境變動。

------

### 🎯 目標 : 利用 paired Monte Carlo simulation，在相同市場路徑與死亡隨機數下比較 No Lapse vs With Lapse
分析：
1. Account Value（AV）
2. Death Benefit（DB）
3. Death Cost（DC）
4. PV Profit（NPV）
5. 拆解利潤來源與 tail risk

----------


#### 💻 核心程式碼 完整模擬過程與數據分析請參考： [GMDB 定價模型主程式 (Jupyter Notebook)](./Simulation.ipynb)
---

### 🧩 Model Framework

### Simulation Flow

```mermaid
graph TD
    %% 主流程
    Input[商品參數設定] --> stochastic[隨機路徑生成]
    stochastic --> Simulation[蒙地卡羅模擬流程]
    Simulation  --> Result[Profit and Risk Metrics]

    %% 蒙地卡羅詳細步驟 (子圖)
    subgraph Simulation [蒙地卡羅模擬流程]
        S1[隨機抽樣死亡判定] -->|U < qx| Dead[死亡: 觸發理賠]
        S1 -->|U >= qx| Alive[生存]
        Dead --> DB[計算給付額DB]
        Alive --> |AV不足扣款| Lapse[保單失效]
        Alive --> |存續|S2[模擬標的資產路徑: GBM 、隨機解約模擬]
        S2 --> |lapse| Lapse2[解約: 計算surrender charge/ surrender value]
        S2 --> |NO lapse|S3[套用保單結構更新AV]
        S3 --> |進入下個月模擬| S1
    end

    style Dead fill:#ffe9ef,stroke:#ffdee7
    style Alive fill:#ffe9ef,stroke:#ffdee7
    style Simulation fill:#c0d9d9,stroke:#01579b,stroke-width:2px
```

</br>

<模型重點>  
🔹 Monte Carlo + GBM : 模擬投資標的隨機路徑 → 產生 AV 分布  
🔹 GMDB 設計 : DB = max(保證金額, 帳戶價值)  
🔹 Death Cost = max(DB - AV, 0) → 代表公司實際承擔的保證損失  
🔹 Paired Simulation : 同一保戶 / 同一市場路徑 / 同一死亡亂數 → 只改「是否解約」  
🔹 Logistic Model : 基礎解約意願 / Moneyness / 解約費用 / 市場報酬 → 隨機亂數<解約機率 視為解約

---
### 📊 模擬結果
#### 1.  No Lapse / With Lapse 模型基本比較
   
- Final AV : With lapse（153萬）低於 No lapse（180萬）(解約後 AV 補 0 → 拉低整體平均)
- Lapse Behavior : With lapse 模型解約率約 9.9%、平均發生於 第 13 年
- Duration :　With lapse 較短（227 vs 235 月） 👉 減少收入，同時降低風險暴露期間
- Death Rate : 7.3% → 6.6%  解約使部分保戶提前退出風險池 👉  Unconditional DB / Death Cost 下降

| Metric                             | No Lapse       | With Lapse     |
|:-----------------------------------|---------------:|---------------:|
| Unconditional Final AV             | 1,805,082.3936 | 1,531,913.0913 |
| Conditional Final AV               | 1,947,230.1980 | 1,834,626.4567 |
| Lapse Rate                         | 0.0000         | 0.0990         |
| Avg Lapse Month                    | --             | 155.8788       |
| Duration                           | 234.9920       | 226.9040       |
| Death Rate                         | 0.0730         | 0.0660         |
| Unconditional Death Benefit (Mean) | 140,013.8139   | 120,646.4503   |
| Unconditional Death Cost (Mean)    | 19,930.2081    | 18,286.4170    |

![圖片描述](Images/av.png)

---

#### 2.  Conditional Death Benefit vs Death Cost

| Metric   | DB (No Lapse)   | DB (With Lapse)   | Death Cost (No Lapse)   | Death Cost (With Lapse)   |
|:---------|----------------:|------------------:|------------------------:|--------------------------:|
| Mean     | 1,917,997.4506  | 1,827,976.5197    | 273,016.5489            | 277,066.9240              |
| Median   | 1,600,000.0000  | 1,600,000.0000    | 186,248.4303            | 200,121.8030              |
| P95      | 3,079,608.0688  | 2,758,799.2435    | 829,880.3162            | 834,639.8409              |
| P99      | 4,406,276.8821  | 3,067,684.4443    | 972,626.5051            | 987,544.0700              |
| Max      | 5,154,733.7489  | 3,250,789.7456    | 1,126,064.3153          | 1,126,064.3153            |


- Death Benefit（給付金額）  
(1) Mean / Median  
With lapse 平均 DB 較低 👉 高 AV 保戶提前解約 → 高給付案例減少  
DB median 皆為 160萬 👉 至少一半案例仍由保證機制主導  

(2) Tail（P95 / P99 / Max）    
With lapse 下 P95、P99、Max 右尾明顯下降 👉 DB tail 來自高 AV（好市場），Lapse 移除高 AV 保戶 → 高給付減少  

- Death Cost（實際風險）      
(1) Mean  
With lapse 略高 👉 解約移除低風險（高 AV）路徑 留下來的死亡案例 AV 較低  

(2) Tail（P95 / P99 / Max）  
With lapse 下P95 / P99：略微上升 Max：兩情境相同 👉 尾端損失未下降

🔥 Core Insight
Death Benefit ↓（高 AV 給付減少）
Death Cost 尾端幾乎不變，未降低真正的保證風險。

---
#### 3.  PV Profit 分析（Log Scale）

| Metric       | No Lapse         | With Lapse       |
|:-------------|-----------------:|-----------------:|
| Min          | -798,481.0257    | -798,481.0257    |
| P1           | -402,261.9864    | -402,261.9864    |
| P5           | 109,459.5953     | 105,271.9762     |
| Median       | 132,054.6660     | 131,398.1018     |
| Mean         | 117,115.3022     | 117,467.4978     |
| Total Profit | 117,115,302.2221 | 117,467,497.8194 |

(1) Min、P1  
在 no lapse 與 with lapse 下皆相同、P5接近 👉 最極端虧損路徑幾乎相同  

(2) Mean、Total PV Profit  
with lapse 平均 profit 略為右移  



🔥 Core Insight
最壞情境（極端虧損）在兩種情境下幾乎相同，代表 lapse 無法消除 joint tail event（市場下跌 + 死亡）。
但左尾（虧損區）幾乎重疊，顯示 tail risk 並未改善。

🔍  Outcome Decomposition (將保戶分為三類)

| case_type              |   n | avg_db_no_lapse   | avg_db_with_lapse   | avg_dc_no_lapse   | avg_dc_with_lapse   | avg_pv_profit_no_lapse   | avg_pv_profit_with_lapse   |
|:-----------------------|----:|------------------:|--------------------:|------------------:|--------------------:|-------------------------:|---------------------------:|
| Same outcome           | 901 | 133,902.8305      | 133,902.8305        | 20,295.6903       | 20,295.6903         | 117,100.8448             | 117,100.8448               |
| Survivor lapsed        |  92 | 0.0000            | 0.0000              | 0.0000            | 0.0000              | 128,653.4550             | 120,959.6084               |
| Death avoided by lapse |   7 | 2,766,766.2280    | 0.0000              | 234,827.2981      | 0.0000              | -32,668.1179             | 118,764.6655               |

1. Same outcome >> (1) 保戶在兩邊皆死亡，且死亡發生在解約前 或 (2) 兩邊都未死亡、with lapse 也沒有解約
這兩種情況兩模型現金流完全一致， PV profit 沒有任何差異。

2. Survivor lapsed 
保戶在兩種情境下皆未死亡( DB & DC = 0)，但 with lapse 下保戶提前解約少收未來管理費與 COI，雖有 surrender charge 作補償，但整體 PV profit 略為下降，顯示對於原本不會產生保證成本的保戶而言，lapse 反而降低公司利潤。

3. Death avoided by lapse 
No lapse 原本死亡保戶在 with lapse 下於死亡前提前解約，完全避免DB & DC ，PV profit 轉為正值（118,765），平均提升約 15 萬。這顯示 lapse 的主要價值並非來自解約費收入，而是來自避免少數高損失的死亡情境。


|類型	|說明	|對 Profit 影響|
|Same outcome	|無差異	|無影響|
|Survivor lapsed	|提前解約	|小幅下降|
|Death avoided by lapse	|避開死亡	|大幅提升|


🔥 Core Insight : 平均 profit 的提升來自少數「避免死亡」的案例，而非整體改善。

## 🔥 核心發現
1️⃣ Lapse 的雙重效果
Fee 收入 ↓
Risk 暴露 ↓
2️⃣ 平均 vs Tail
平均 Profit ↑
Tail Risk ≈ 不變

👉 關鍵原因：

Lapse reduces frequency, not severity
3️⃣ 風險來源

GMDB 的主要風險來自：

市場大跌 + 發生死亡（Joint Tail Event）
4️⃣ 商品設計含意
Lapse 是「風險釋放機制」
不是「風險解決機制」

👉 Tail risk 仍需：

保證設計
費率調整
Hedging
⚠️ 模型限制
單一資產（GBM）/ 年限 20 年
無 stochastic interest rate
無 hedging cost
lapse 未校準

