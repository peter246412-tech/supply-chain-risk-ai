# Supply-Signal-AI
AI-based supply chain risk prediction using market, news, and logistics data

## Overview

SSAI is a project that designs a supply-chain risk scoring system for semiconductor-related mid-sized companies.

The main idea is simple: supply-chain risks usually do not appear suddenly. Before a disruption becomes serious, small signals often appear in different places such as market prices, news articles, geopolitical events, and logistics data. However, these signals are usually checked separately, so it is difficult for a company to understand the overall risk level quickly.

This project tries to combine those signals into one explainable risk score.

## Problem

Mid-sized companies often do not have a separate risk analysis team. A purchasing or procurement manager may check news, exchange rates, material prices, and logistics issues, but these data are scattered.

Because of this, decision-making often happens after the problem has already become serious.

The key problem of this project is:

> How can we help mid-sized semiconductor-related companies interpret scattered supply-chain signals earlier and more clearly?

## Target Users

The target users are procurement or purchasing managers in Tier 1–2 semiconductor-related mid-sized firms.

They need a simple system that can answer three questions:

1. Is the supply-chain risk getting higher?
2. What is causing the risk?
3. What action should the company consider?

## Data Structure

SSAI uses four main data groups: market, news, geopolitical, and logistics data.

### 1. Market Data

Market data is used to capture cost pressure and price instability.

Example variables:

| Variable              | Meaning                        | Example Source       |
| --------------------- | ------------------------------ | -------------------- |
| USD/KRW exchange rate | Import cost pressure           | Bank of Korea ECOS   |
| WTI oil price         | Energy and transportation cost | EIA                  |
| Copper price          | Material cost pressure         | LME or financial API |
| Import price index    | General import cost trend      | Bank of Korea ECOS   |
| Export price index    | External price trend           | Bank of Korea ECOS   |

Example table structure:

| Column        | Description                                  |
| ------------- | -------------------------------------------- |
| date          | Observation date                             |
| source        | Data source name                             |
| variable_name | Name of indicator, such as USD_KRW or COPPER |
| value         | Numeric value                                |
| unit          | Unit of measurement                          |
| collected_at  | Time when the data was collected             |

Main features calculated from this data:

* 7-day change rate
* 30-day change rate
* 30-day volatility
* sudden spike detection
* moving average deviation

These features are used to calculate the market risk score.

### 2. News Data

News data is used because news often reflects early signals of supply-chain problems.

News articles are collected using keyword combinations related to:

| Group     | Example Keywords                                    |
| --------- | --------------------------------------------------- |
| Industry  | semiconductor, chip, foundry, packaging             |
| Resources | copper, helium, neon, rare earth                    |
| Countries | Taiwan, China, United States, Japan, Korea          |
| Events    | export control, shortage, delay, regulation, strike |

Example raw news table:

| Column        | Description                      |
| ------------- | -------------------------------- |
| article_id    | Unique article ID                |
| title         | News title                       |
| body          | Article text                     |
| published_at  | Published date                   |
| source_name   | Media company                    |
| url           | Article URL                      |
| keyword_query | Keyword used for collection      |
| collected_at  | Collection time                  |
| body_hash     | Hash value for duplicate removal |

After collection, each article is processed and tagged.

Example analyzed news structure:

| Column          | Description                                         |
| --------------- | --------------------------------------------------- |
| article_id      | Article ID                                          |
| relevance_score | How relevant the article is to supply-chain risk    |
| risk_type       | Market, production, logistics, regulation, or mixed |
| severity_score  | Estimated seriousness of the risk                   |
| country_tags    | Related countries                                   |
| resource_tags   | Related materials or resources                      |
| event_tags      | Related risk events                                 |

For the first version, keyword-based tagging can be used. In a later version, a KLUE-BERT model can be used to classify article relevance, risk type, severity, and entities.

### 3. Geopolitical Data

Geopolitical data is used to represent structural dependency.

Unlike news data, this part does not change every day. It is more like a background risk setting.

Example structure:

| Country       | Main Risk Type              | Example Meaning                           |
| ------------- | --------------------------- | ----------------------------------------- |
| Taiwan        | Production risk             | Foundry concentration                     |
| China         | Logistics and assembly risk | Supply flow and assembly dependency       |
| United States | Regulation risk             | Export control and technology restriction |
| Japan         | Material risk               | Semiconductor material dependency         |

Example YAML-style structure:

```yaml
countries:
  taiwan:
    production_dependency: 0.40
    risk_weight: 0.40
    type: production

  china:
    logistics_dependency: 0.30
    risk_weight: 0.30
    type: supply_and_assembly

  usa:
    regulation_dependency: 0.30
    risk_weight: 0.30
    type: export_control
```

This structure is combined with news-based event intensity.
For example, if China-related regulation news increases, the China geopolitical risk score becomes higher.

### 4. Logistics Data

Logistics data is used to check whether the actual flow of goods is becoming unstable.

Example variables:

| Variable          | Meaning                                |
| ----------------- | -------------------------------------- |
| Port cargo volume | Amount of cargo handled by major ports |
| Container volume  | Container traffic volume               |
| Vessel count      | Number of ship arrivals and departures |
| BDI               | Global shipping cost pressure          |
| SCFI              | Container freight cost indicator       |

Example logistics table:

| Column           | Description                             |
| ---------------- | --------------------------------------- |
| port_name        | Name of port, such as Busan or Shanghai |
| date             | Observation date                        |
| vessel_count     | Number of vessels                       |
| cargo_volume     | Cargo volume                            |
| container_volume | Container volume                        |
| shipping_index   | BDI or SCFI value                       |
| index_type       | Type of shipping index                  |
| collected_at     | Collection time                         |

Main features calculated from logistics data:

* month-over-month cargo volume decrease
* number of consecutive months with decreasing volume
* port concentration index
* shipping index deviation from moving average

These features are used to calculate the logistics risk score.

## Scoring Logic

Each data group is converted into a partial score between 0 and 100.

The final score is calculated as:

```text
Risk Score = 0.30 × Market Score
           + 0.30 × News Score
           + 0.20 × Geopolitical Score
           + 0.20 × Logistics Score
```

The reason market and news have higher weights is that they can show early warning signals faster than structural or monthly logistics data.

## Risk Level

The final score is converted into a simple risk level.

| Score Range | Risk Level | Meaning                       |
| ----------- | ---------- | ----------------------------- |
| 0–30        | Stable     | Normal monitoring             |
| 31–60       | Caution    | Some warning signals exist    |
| 61–80       | Risk       | Multiple indicators show risk |
| 81–100      | High Risk  | Immediate response is needed  |

## Example Output

```json
{
  "risk_score": 78,
  "risk_level": "Risk",
  "score_breakdown": {
    "market": 65,
    "news": 82,
    "geopolitical": 74,
    "logistics": 71
  },
  "top_causes": [
    "Helium shortage-related news has increased recently.",
    "China-related export control news is concentrated.",
    "Shanghai port cargo volume decreased compared to the previous month."
  ],
  "recommended_actions": [
    "Check inventory of key process gases.",
    "Review components dependent on China.",
    "Consider alternative logistics routes."
  ]
}
```

## Expected System Flow

1. Collect raw data from APIs, news sources, and public statistics
2. Store raw data without modification
3. Clean and preprocess each data type
4. Create features such as change rate, volatility, frequency, and severity
5. Convert each domain into a 0–100 risk score
6. Combine the scores into one final risk score
7. Generate explanations and recommended actions

## Dashboard Design

The system is designed to be used through a simple dashboard rather than raw data outputs. The goal is to help users quickly understand the current risk situation without needing deep technical knowledge.

The dashboard is composed of the following components:

* **Risk Gauge**
  A visual indicator showing the current risk score (0–100) with color-coded levels (green, yellow, orange, red).

* **Domain Breakdown (Radar Chart)**
  Displays the four domain scores (Market, News, Geopolitical, Logistics) so users can see which factor is driving the risk.

* **Trend Analysis (Time Series Chart)**
  Shows how the risk score has changed over time (e.g., last 30 days), allowing users to identify increasing trends.

* **Top Risk Factors**
  Lists the main causes contributing to the current risk score (e.g., increase in negative news, port congestion).

* **Recommended Actions Panel**
  Provides simple and actionable suggestions based on the detected risk factors.

* **Risk-related News Feed**
  Displays recent articles related to supply-chain risks, including risk type and severity.

The dashboard is designed to answer three key questions:

1. How risky is the situation now?
2. What is causing the risk?
3. What should we do next?

---

## Business Value and Solutions

The main purpose of SSAI is not just to analyze data, but to support real-world decision-making.

Based on the risk score and its causes, the system suggests practical actions for companies:

* **Inventory Strategy Optimization**
  Instead of increasing inventory for all items, companies can focus on high-risk materials and selectively secure safety stock.

* **Alternative Supplier Identification**
  If risk is concentrated in a specific country or supplier, the system encourages early exploration of alternative sources.

* **Dependency Adjustment**
  Helps companies identify over-dependence on certain countries, materials, or logistics routes and gradually diversify them.

* **Faster Risk Reporting**
  Provides a structured way to report risks to management using clear scores and explanations instead of vague descriptions.

* **Early Response Timing**
  The biggest value is enabling action before the disruption becomes severe, rather than reacting after it happens.

Overall, SSAI works as a lightweight decision-support tool for companies that do not have a dedicated risk analysis team.

---

## Scalability and Future Business Expansion

SSAI is designed not only as a one-time analysis tool but as a scalable service.

### 1. Subscription-based Model

The system can be offered as a low-cost SaaS (Software-as-a-Service) solution for mid-sized companies.

Possible features:

* Daily risk score monitoring
* Industry-specific dashboards
* Custom alerts for specific materials or countries

---

### 2. Custom Risk Monitoring

Different companies have different supply-chain structures.
SSAI can be extended to support:

* Company-specific dependency settings
* Customized weighting of risk factors
* Tailored risk reports for specific products or components

---

### 3. Industry Expansion

Although this project focuses on semiconductors, the same framework can be applied to other industries:

* Automotive supply chains
* Energy and raw materials
* Global manufacturing networks

The core idea (integrating dispersed signals into one explainable score) remains the same.

---

### 4. Advanced AI Integration

Future improvements may include:

* Real-time data pipelines
* More advanced NLP models for news analysis
* Graph-based supply-chain modeling
* Predictive simulation for “what-if” scenarios

---

### 5. Enterprise-level Features

For larger companies, the system can be expanded into:

* Integration with ERP systems
* Automated executive reports
* Risk alert APIs
* Multi-region monitoring dashboards

---

## Final Note

The long-term goal of SSAI is not to perfectly predict the future, but to reduce uncertainty and improve the probability of making better decisions under risk.


## Tech Stack

* Python
* pandas / numpy
* scikit-learn
* PostgreSQL
* FastAPI
* Streamlit or React
* KLUE-BERT for future news classification

## Current Status

This repository currently contains the concept design and system specification documents.
The next step is to implement a rule-based MVP using public data and then improve the news module with a Korean language model.

## Author

Jinwoo Song
Social Science & Artificial Intelligence
