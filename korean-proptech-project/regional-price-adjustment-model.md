# Regional Price Adjustment Model Based on Apartment Sales Price Index

## Table of Contents
1. [Background and Objectives](#1-background-and-objectives)
2. [Price Adjustment Algorithm](#2-price-adjustment-algorithm)
3. [Regional Price Index Analysis](#3-regional-price-index-analysis-as-of-march-2025)
4. [Calculation Examples](#4-calculation-examples)
5. [Data-Driven Key Insights](#5-data-driven-key-insights)
6. [Why Index-Based Adjustment is Essential](#6-why-index-based-adjustment-is-essential-the-why)
7. [Use Cases](#7-use-cases)
8. [Limitations and Precautions](#8-limitations-and-precautions)

## 1. Background and Objectives
Official assessed values, which serve as the foundation for real estate asset valuation, are calculated annually and thus cannot reflect the time lag of rapidly changing markets, nor can they capture the varying recovery elasticity across different regions.

This document aims to introduce a model that precisely adjusts the gap between official assessed values and actual market prices using the latest Apartment Sales Price Index (as of March 2025) from the Korea Real Estate Board, based on data-driven methodology.

**Expected Benefits:**
- Real-time reflection of market volatility
- Precise capture of regional price disparities
- Enhanced accuracy in asset valuation

## 2. Price Adjustment Algorithm

### 2.1 Core Formula

Current market price is derived by multiplying the official assessed value by a region-specific weight (W).
```
W = (1 ÷ R) × (I_current ÷ I_appraisal)
```

- **R (Realization Rate)**: Fixed at 0.8 (80%), base multiplier 1.25
- **I_current**: Current sales price index
- **I_appraisal**: Sales price index at the appraisal base date

### 2.2 Calculation Methodology

This model adopts an approach that **adjusts only the pure market fluctuations after the appraisal date**.

- Official assessed values are based on market prices at a specific point in time (e.g., January 2024)
- Market fluctuation to date = I_current ÷ I_appraisal
- This is combined with the reciprocal of the realization rate (1.25) to calculate the final adjustment factor

**Advantages:**
- ✅ Adjusts only for time-lag after appraisal
- ✅ Prevents excessive weight calculation due to long-term cumulative effects
- ✅ Sustainable model even 10 years later

## 3. Regional Price Index Analysis (as of March 2025)

Analysis of Korea Real Estate Board data reveals an extreme polarization phenomenon where adjustment weights vary dramatically even among districts within Seoul.

| Category | Region | Sales Price Index (Mar 2025) | Change vs Jun 2021 | Data Insight |
|----------|--------|------------------------------|---------------------|--------------|
| **Leading Growth** | Seocho-gu | 117.3 | +17.3% | Highest growth rate in Seoul |
| | Songpa-gu | 113.0 | +13.0% | Strong recovery in Gangnam-3 districts |
| | Gangnam-gu | 112.9 | +12.9% | Sustained upward trajectory |
| **Average/Stable** | Yangcheon-gu | 101.4 | +1.4% | Recovered to 2021 baseline level |
| | Seoul (Overall) | 99.7 | -0.3% | Seoul average near baseline |
| **Stagnant/Declining** | Gangbuk-gu | 88.1 | -11.9% | Approx. 12% below baseline |
| | Dobong-gu | 86.0 | -14.0% | Lowest index in Seoul, continued decline |
| **Risk/Depressed** | Daegu Metro | 75.8 | -24.2% | Significant decline vs 2021 |
| | Sejong City | 70.8 | -29.2% | Lowest index nationwide |

**Note:** For actual weight calculation, use the index at the property's appraisal base date.

## 4. Calculation Examples

Below are examples of adjusting current market prices (March 2025) for apartments with a 2025 official assessed value (based on January 2024) of 1 billion KRW.

### Example 1: Seocho-gu Apartment
```
Official Assessed Value: 1 billion KRW (2025 assessment)
Appraisal Base Date: January 2024 (Index 110.0)
Current Date: March 2025 (Index 117.3)

Weight W = (1 ÷ 0.8) × (117.3 ÷ 110.0) = 1.25 × 1.066 = 1.333
Adjusted Price = 1 billion KRW × 1.333 = 1.33 billion KRW

✅ Interpretation: Reflects ~6.6% increase since appraisal date
```

### Example 2: Yangcheon-gu Apartment
```
Official Assessed Value: 1 billion KRW (2025 assessment)
Appraisal Base Date: January 2024 (Index 100.0)
Current Date: March 2025 (Index 101.4)

Weight W = (1 ÷ 0.8) × (101.4 ÷ 100.0) = 1.25 × 1.014 = 1.268
Adjusted Price = 1 billion KRW × 1.268 = 1.27 billion KRW

✅ Interpretation: Reflects ~1.4% increase since appraisal date
```

### Example 3: Sejong City Apartment (Declining Trend)
```
Official Assessed Value: 1 billion KRW (2025 assessment)
Appraisal Base Date: January 2024 (Index 72.0)
Current Date: March 2025 (Index 70.8)

Weight W = (1 ÷ 0.8) × (70.8 ÷ 72.0) = 1.25 × 0.983 = 1.229
Adjusted Price = 1 billion KRW × 1.229 = 1.23 billion KRW

⚠️ Interpretation: Reflects ~1.7% decline since appraisal date
```

**Key Differences (Before vs After Improvement):**
- **Before**: Reflected all cumulative changes from 2021 → Seocho 1.47B, Sejong 0.89B (excessive gap)
- **After**: Reflects only changes since appraisal → Seocho 1.33B, Sejong 1.23B (reasonable gap)
- Regional disparities now reflect only post-appraisal market movements, making them far more realistic

## 5. Data-Driven Key Insights

1. **Stark Contrast in Regional Recovery Speeds**: As of March 2025, Seocho-gu (117.3) has risen 17.3% from 2021, while Sejong City (70.8) has fallen 29.2%, showing a 46.5-point gap. This indicates completely different market dynamics across regions.

2. **Short-term Momentum Differences**: Over the past 6 months (Sep 2024 - Mar 2025), Seocho-gu surged from 111.6 to 117.3 (+5.1%), while Dobong-gu stagnated from 85.9 to 86.0 (+0.1%), widening the regional recovery speed gap.

3. **Rise of Regional Powerhouses**: Beyond Seoul's Gangnam area, certain regional cities like Sangju-si in Gyeongsangbuk-do (117.0) and Jecheon-si in Chungcheongbuk-do (114.9) record indices comparable to Seocho-gu, indicating selective regional growth rather than concentration in the capital area.

4. **Persistent Depression in Declining Regions**: Areas like Sejong (70.8) and Daegu (75.8) have been continuously declining since 2021, and if additional declines occur after appraisal, adjusted prices may fall below official assessed values.

## 6. Why Index-Based Adjustment is Essential (The Why)

1. **Time-Lag Correction**
    - Official assessed values are based on market prices at a past specific date (e.g., January 2024)
    - Sales price indices track the latest market movements on a monthly basis
    - Reflecting changes from appraisal date to present precisely **corrects price distortions due to time lag**

2. **Precise Reflection of Regional Gradient**
    - During the same period (Jan 2024 - Mar 2025), Seocho-gu rose +6.6% while Sejong City fell -1.7%, showing completely different regional market dynamics
    - Rather than uniform nationwide adjustment, this enables accurate asset valuation by reflecting **region-specific market flows**

3. **Data-Driven Objective Adjustment**
    - Based on official statistics from Korea Real Estate Board, enabling **mathematically grounded price calculation** free from subjective judgment
    - Establishes a transparent and reproducible valuation process

4. **Long-term Sustainability**
    - By adjusting based on appraisal date, weights do not become excessively large even 10 years later
    - Model can be used continuously without distortion from cumulative effects

## 7. Use Cases

This model can be applied to the following practical domains:

### 7.1 Asset Management and Portfolio Valuation
- **Real-time valuation of real estate asset portfolios**: Transform static appraisal-based valuation into dynamic market index-based valuation
- **Investment performance monitoring**: Track asset value fluctuation trends by monitoring regional sales index changes

### 7.2 Risk Management
- **Early warning for price reversal regions**: Set risk flags for regions with indices below 80
- **Regional volatility analysis**: Assess market stability through 6-month and 1-year index change rates

### 7.3 Collateral Valuation
- **Utilization in loan collateral assessment**: Calculate actual collateral value by applying regional weights to official assessed values
- **LTV (Loan to Value) recalculation**: Set accurate loan limits reflecting market volatility

### 7.4 Decision Support
- **Buy/sell decisions**: Determine optimal transaction timing considering regional market trends
- **Regional selection strategy**: Comparative analysis of leading growth regions vs undervalued regions

## 8. Limitations and Precautions

When applying this model in practice, the following must be considered:

### 8.1 Structural Limitations of the Model

**✅ Improvements Made:**
- **Fixed Baseline Problem Resolved**: Previously, the fixed 2021 baseline caused weights to become unrealistically large over time due to cumulative increases. The improved model uses the appraisal date as baseline, resolving this issue.

**⚠️ Remaining Limitations:**
1. **Individual Property Characteristics Not Reflected**
   - This model uses regional average indices, so micro-factors like apartment brand, floor level, orientation, and area are not reflected.
   - Actual market prices of individual properties within the same region may differ by ±20% or more.

2. **Fixed Realization Rate**
   - Uses a fixed 80% realization rate, but in reality, realization rates may vary by region and property type.
   - Model recalibration needed if official assessed value realization rate policy changes after 2024.

3. **Limited to Apartments**
   - This model is based on Korea Real Estate Board's **Apartment Sales Price Index**, making direct application to officetels, villas, and detached houses difficult.

### 8.2 Data Operational Precautions
1. **Appraisal Base Date Verification Required** ⭐
   - To apply this model, you **must verify the appraisal base date** of the property.
   - Accurate identification of the sales price index (I_appraisal) at the appraisal base date is essential for correct weight calculation.
   - While official assessed values are generally based on January 1 of the previous year, this may vary by individual property, requiring verification.

2. **Monthly Index Updates Required**
   - Sales price indices are updated monthly, requiring periodic updates with the latest data for accurate adjustment.
   - Data delays may cause 1-2 months of lag.

3. **Limitations During Rapid Market Changes**
   - During rapid market fluctuations due to policy changes, interest rate spikes, economic crises, etc., there may be lags in index reflection.

4. **Regional Granularity Level**
   - Cannot capture price differences at units (dong/unit number) more granular than the index-provided regional units (si/gun/gu).

### 8.3 Usage Recommendations
- **Use as Supplementary Indicator**: Do not use this model as the sole valuation standard; cross-reference with actual transaction prices, asking prices, and professional appraisals
- **Expert Verification**: For critical decisions (large-scale investments, collateral settings), always require verification by real estate professionals
- **Continuous Monitoring**: Periodically validate model performance and track error rates against actual transaction prices for improvement

---

**Written**: March 2025
**Data Source**: Korea Real Estate Board Apartment Sales Price Index
