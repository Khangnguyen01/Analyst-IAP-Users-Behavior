# IAP Users – Limited Ads Experiment (Notebook) 🎮📊

Understand how **limiting rewarded ads** right after a user’s **first in‑app purchase (IAP)** affects **retention and revenue** — and decide **how long** to keep the limit before letting them watch ads for resources again.

This README documents the analysis notebook: **`IAP Users Behavior.ipynb`**.
---
## 🎯 Objective

You're running a test where **every user’s first IAP** triggers an **ad‑limit** (cap or cooldown) on rewarded ads.  
The notebook helps you answer:

1) **What exactly is the ad limitation** (cap/cooldown) users experience post‑IAP?  
2) **How long** should we keep ads limited before allowing rewarded ads again?  
3) Does relaxing the limit at a certain time **increase retention** without hurting **monetization**?

---

## 📈 Metrics & Guardrails

- **Retention**: D1/D3/D7/D14 (by cohort of *IAP day 0*), rolling 7‑day too.
- **ARPDAU**: Ads + IAP combined.
- **ARPPU** (IAP only) & **Ads ARPDAU** (ads only).  
- **Next‑Purchase Delay**: days from IAP1 → next IAP.  
- **Ads Consumption**: rewarded ads per user/day (pre/post limit).  
- **Cannibalization Ratio**: `ΔIAP / (−ΔAds)` (is lost ads revenue offset by IAP gain?).  
- **Resource Inflation Risk**: resources per user/day via ads vs sinks.

Guardrails to monitor: **crash rate**, **levels/time played**, **FTUE completion**, **negative reviews** spikes.

---

## 🧪 Experiment Design (concepts used in the notebook)

- **Cohort** = users whose **first IAP** falls on date *t0* (UTC).  
- **Treatment** = post‑IAP **ad limitation** (either *cooldown days* or *daily cap*).  
- **Comparison** = users with lighter/no limitations *or* synthetic counterfactual by modeling (see below).  
- **Outcome windows** = `t0+1 … t0+N` days for retention; `t0+N` for monetization summaries.

**Analysis toolkit inside the notebook** (based on headings & imports):
- **BigQuery** extraction (SQL templates provided below)
- **Pandas** transforms & **Seaborn/Matplotlib** viz
- **Retentioneering / Eventstream** for step‑flow visualizations
- Survival curves (time‑to‑churn), *hazard vs cooldown length* comparisons

---

## 🗂️ Data Requirements

Assumed event schema (GA4‑like, adapt if different):

- `event_name` ∈ {`user_engagement`, `ad_reward`, `ad_impression`, `iap_purchase`}  
- `event_timestamp` (microseconds)
- `user_pseudo_id` (or user_id)
- `value` (IAP revenue), `ad_value` (ads revenue if available)
- Optional: `level`, `country`, `platform`, `session_id`

**Derived fields in notebook**
- **IAP1_date** per user (first `iap_purchase`)
- **Cooldown / Cap exposure** per user/day  
- **Retention flags** (`active_d+1`, `active_d+7`, …)

---

## 🧾 BigQuery – Starter SQL (edit in the first cells)

```sql
-- Cohort: first purchasers
WITH iap1 AS (
  SELECT
    user_pseudo_id,
    TIMESTAMP_MICROS(MIN(event_timestamp)) AS iap1_ts,
    DATE(TIMESTAMP_MICROS(MIN(event_timestamp))) AS iap1_date
  FROM `project.dataset.events_*`
  WHERE event_name = 'iap_purchase'
  GROUP BY 1
),
ads AS (
  SELECT
    user_pseudo_id,
    DATE(TIMESTAMP_MICROS(event_timestamp)) AS event_date,
    COUNTIF(event_name = 'ad_reward') AS rewarded_ads,
    SUM(ad_value) AS ads_revenue
  FROM `project.dataset.events_*`
  WHERE event_name IN ('ad_reward','ad_impression')
  GROUP BY 1,2
),
act AS (
  SELECT
    user_pseudo_id,
    DATE(TIMESTAMP_MICROS(event_timestamp)) AS event_date,
    1 AS active
  FROM `project.dataset.events_*`
  WHERE event_name IN ('user_engagement','session_start','level_start')
)
SELECT
  i.user_pseudo_id,
  i.iap1_date,
  a.event_date,
  a.rewarded_ads,
  a.ads_revenue,
  IFNULL(t.active,0) AS active_flag
FROM iap1 i
LEFT JOIN ads a ON a.user_pseudo_id = i.user_pseudo_id
LEFT JOIN (
  SELECT user_pseudo_id, event_date, MAX(active) active
  FROM act GROUP BY 1,2
) t USING (user_pseudo_id, event_date);
```

> Add **limit policy** metadata (cap/day or cooldown days) to compute exposure per user/day.

---

## 🔧 Notebook Flow (what each section does)

1. **Parameters**: policy type (`cooldown_days` *or* `cap_per_day`), test start/end, markets, platforms.
2. **Data Pull** (BigQuery): fetch cohort, daily activity, ads usage, revenues.
3. **Feature Build**:
   - **Exposure**: for each user/day, compute if under limitation (1/0) and the **remaining cap**.
   - **Outcomes**: retention flags, IAP revenue, ads revenue.
4. **Visuals & EDA**:
   - **Step Matrix / Sankey**: IAP1 → {Play / WatchAd / NoAd / Churn}.
   - **Survival Curves**: time to churn by cooldown length.
   - **Dose–Response**: retention vs number of limited days, retention vs cap.
5. **Counterfactual / Uplift** (lightweight):
   - Stratify by country/platform/level‑band to reduce bias.
   - Compare matched strata across cooldown choices or caps.
6. **Policy Sweep**:
   - Try **cooldown_days ∈ {0,1,3,5,7,14}** and/or **cap ∈ {0,1,3,5}**.
   - Produce a table: **Retention Δ**, **Ads Δ**, **IAP Δ**, **ARPDAU Δ**, **Net Δ**.
7. **Recommendations**:
   - Pick the **Pareto** policy (max retention with non‑negative net revenue).  
   - Output a human‑readable summary & CSV.

---

## 📤 Outputs

- `fig/` **Retention curves**, **ads per user/day**, **step flows**
- `tables/` policy sweep with **ΔRetention**, **ΔIAP**, **ΔAds**, **Net**
- `export/` recommended policy note (Markdown) + CSV for BI

Folder names are configurable in the **Paths** cell.

---

## 🧭 How to Run (Colab/Jupyter)

1. Open **`IAP Users Behavior.ipynb`** in Colab or Jupyter.
2. Set **Parameters** (policy, dates, markets, paths).
3. Run **Data Pull** → **Feature Build** → **Visuals** → **Policy Sweep**.
4. Read the **Recommendations** cell and download artifacts from `export/`.

> If you do not have direct BigQuery access from Colab, use a service account JSON and `gcloud auth application-default login`, or export the base tables then upload CSVs.

---

## 🧠 Interpreting Results

- **Short cooldown (≤3 days)** often unlocks **early retention** gains with minimal ads loss.  
- **Long cooldowns** can increase IAP but risk higher churn if players rely on ad resources.  
- A **small daily cap** (e.g., 3–5 ads) during cooldown can balance resource flow and satisfaction.

> Use the **policy sweep table** to locate the **knee point** where retention improvement starts flattening but monetization remains safe.

---

## ⚠️ Pitfalls

- **Selection bias**: high‑value users may behave differently; stratify or control for spend/level.  
- **Late attribution**: IAP posts can lag; include a **lookback window**.  
- **Event gaps**: use sessions/engagement as backup for activity.  
- **Resource inflation**: if sinks are weak, ads can dominate progression; monitor **economy sinks**.

---

## 📌 FAQ

**Q: Can I reuse this for non‑IAP cohorts?** Yes — swap the cohort definition to `first_ad_watchers` or `first_session`.

**Q: Can I compare countries/platforms?** Yes — parameters include filters; the sweep aggregates by segment too.

**Q: Can I push outputs to BigQuery/Looker?** Export CSVs and ingest to your BI pipeline; schema is included in the `tables/` CSV headers.

---

## 🔒 Access & Roles

- **BigQuery**: `roles/bigquery.dataViewer` on source datasets; `roles/bigquery.jobUser` on project.  
- Ensure PII policies are respected; use hashed user IDs if sharing charts externally.

---

## 🗺️ Roadmap (suggested)

- Propensity model (XGBoost/CausalForest) for **uplift** per user segment.  
- Bayesian optimization over **cooldown**/**cap** with weekly re‑tuning.  
- In‑game **A/B switch** for live holdouts.

