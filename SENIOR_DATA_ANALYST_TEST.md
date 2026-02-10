# Senior Data Analyst Test Response

This document outlines how I would approach a senior-level data analyst assessment: clarifying goals, validating data, writing SQL for common product analytics tasks, and describing experimentation and investigation playbooks.

## 1) Problem framing
- Confirm the business question (e.g., improve activation, increase revenue, reduce churn) and the primary metric(s) that represent success.
- Define the unit of analysis (user, session, order) and the time window for evaluation.
- Identify stakeholders and how the output will be consumed (dashboard, memo, ad-hoc query).

## 2) Data validation checklist
- Schema sanity: column types, primary/foreign keys, uniqueness of IDs, and time zone consistency.
- Completeness: missing dates, null / empty strings, late-arriving data, and gaps in event pipelines.
- Consistency: session/order counts across fact tables, duplicated events, and referential integrity to dimension tables.
- Outliers and bots: impossible timestamps, negative revenues/quantities, and abnormal burst activity.
- Reconcile aggregates to source-of-truth reports (finance, billing, web analytics) before proceeding.

## 3) Core metrics and SQL examples
Assume a warehouse with these simplified tables:
- `users(user_id, created_at, country)`
- `events(user_id, event_date, event_type, platform)` with `event_type` values like `view`, `add_to_cart`, `purchase`
- `orders(order_id, user_id, order_date, revenue)`

### 3.1 Daily active users (last 30 days, by platform)
```sql
SELECT
  event_date,
  platform,
  COUNT(DISTINCT user_id) AS dau
FROM events
WHERE event_date >= CURRENT_DATE - INTERVAL '30 day'
GROUP BY event_date, platform
ORDER BY event_date, platform;
```

### 3.2 7-day retention by signup cohort
```sql
WITH cohorts AS (
  SELECT user_id, DATE(created_at) AS signup_date
  FROM users
),
activity AS (
  SELECT DISTINCT user_id, event_date
  FROM events
)
SELECT
  c.signup_date,
  COUNT(DISTINCT c.user_id) AS cohort_size,
  COUNT(DISTINCT CASE WHEN a.event_date BETWEEN c.signup_date + INTERVAL '1 day'
                                    AND c.signup_date + INTERVAL '7 day'
                      THEN a.user_id END) AS retained_d7
FROM cohorts c
LEFT JOIN activity a ON a.user_id = c.user_id
GROUP BY c.signup_date
ORDER BY c.signup_date;
```

### 3.3 Funnel: view → add_to_cart → purchase (last 14 days)
```sql
WITH base AS (
  SELECT user_id, event_type
  FROM events
  WHERE event_date >= CURRENT_DATE - INTERVAL '14 day'
)
SELECT
  COUNT(DISTINCT CASE WHEN event_type = 'view' THEN user_id END)        AS viewers,
  COUNT(DISTINCT CASE WHEN event_type = 'add_to_cart' THEN user_id END) AS adders,
  COUNT(DISTINCT CASE WHEN event_type = 'purchase' THEN user_id END)    AS purchasers;
```

### 3.4 Revenue per user in first 90 days (proxy LTV)
```sql
WITH first_orders AS (
  SELECT
    u.user_id,
    o.order_date,
    o.revenue,
    DATE(u.created_at) AS signup_date
  FROM users u
  JOIN orders o ON o.user_id = u.user_id
  WHERE o.order_date <= u.created_at + INTERVAL '90 day'
)
SELECT
  signup_date,
  COUNT(DISTINCT user_id) AS users_in_cohort,
  SUM(revenue) AS revenue_90d,
  SUM(revenue) / COUNT(DISTINCT user_id) AS arpu_90d
FROM first_orders
GROUP BY signup_date
ORDER BY signup_date;
```

## 4) Experimentation approach (A/B example)
- **Hypotheses:** e.g., New checkout flow increases purchase conversion without hurting AOV.
- **Success metric:** purchase conversion rate; **guardrails:** page load time, refund rate.
- **Sample size:** for proportions, use a two-proportion power calc  
  `n_per_group ≈ 2 * (Z_(1-α/2) + Z_(1-β))^2 * p*(1-p) / Δ^2`, where `p` is baseline rate and `Δ` the minimum detectable effect.
- **Randomization/segmentation:** ensure bucketing by user_id, monitor balance across key dimensions (device, country, traffic source).
- **Analysis:** compute lift, CIs (Wald / Wilson), and a two-sided z-test or chi-square. Validate CUPED or pre-period adjustment if traffic is volatile. Check variance inflation from bots or repeats.
- **Readout:** summarize effect size, significance, practical impact, and recommendation with caveats.

## 5) Metric drop investigation playbook
1. Verify the drop is real: rule out tracking outages, deploys, and data delays.
2. Slice by dimensions (country, device, traffic source, app version) to localize impact.
3. Trace upstream funnel steps to find where the breakage begins.
4. Correlate with recent releases/experiments; rollback or disable treatments if isolated.
5. Quantify revenue/user impact and communicate mitigation steps and owners.

## 6) Deliverables
- SQL notebooks or reproducible queries with comments.
- A concise memo summarizing findings, caveats, and next steps.
- (If ongoing) a dashboard tracking the defined metrics with anomaly alerts.
