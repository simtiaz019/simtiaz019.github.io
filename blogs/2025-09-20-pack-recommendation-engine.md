# Building a Bangladeshi Telco Pack Recommendation Engine with MySQL HeatWave + In-Database LLM

Bangladesh's mobile operators fight churn and grow ARPU by pairing users with the right short-lived data, voice, or combo packs. This guide walks through a production-ready workflow where MySQL HeatWave AutoML learns purchase intent directly from telco telemetry and HeatWave's in-database LLM explains each recommendation — all without exporting data.

## 1. Model the subscriber, pack, and usage universe
Start by provisioning a working schema and domain tables. The script below creates subscriber, pack catalog, usage, and purchase fact tables tailored to BD operators (Grameenphone, Robi, Banglalink):

```sql
CREATE DATABASE IF NOT EXISTS telco_reco;
USE telco_reco;

CREATE TABLE subscribers (
  user_id       VARCHAR(16) PRIMARY KEY,
  msisdn        VARCHAR(20) UNIQUE,
  division      ENUM('Dhaka','Chattogram','Rajshahi','Khulna','Barishal','Sylhet','Rangpur','Mymensingh'),
  age           INT,
  sim_age_days  INT,
  device_tier   ENUM('low','mid','high'),
  arpu_bdt      DECIMAL(10,2)
);

CREATE TABLE packs (
  pack_id     VARCHAR(8) PRIMARY KEY,
  operator    ENUM('GP','Robi','Banglalink'),
  slug        VARCHAR(64) UNIQUE,
  category    ENUM('data','voice','combo','social'),
  data_mb     INT,
  minutes     INT,
  validity_d  INT,
  price_bdt   DECIMAL(10,2),
  description TEXT
);

CREATE TABLE daily_usage (
  user_id     VARCHAR(16),
  usage_date  DATE,
  data_mb     INT,
  voice_min   INT,
  night_pct   TINYINT,
  yt_pct      TINYINT,
  fb_pct      TINYINT,
  wa_pct      TINYINT,
  PRIMARY KEY(user_id, usage_date)
);

CREATE TABLE pack_purchases (
  user_id     VARCHAR(16),
  pack_id     VARCHAR(8),
  purchase_ts DATETIME,
  price_bdt   DECIMAL(10,2),
  PRIMARY KEY(user_id, pack_id, purchase_ts)
);
```

## Resources
- [Subscribers CSV](/asset/blogs/pack_recomm/subscribers.csv)
- [Daily Usage CSV](/asset/blogs/pack_recomm/daily_usage.csv)
- [Pack Purchases CSV](/asset/blogs/pack_recomm/pack_purchases.csv)
- [Packs Catalog CSV](/asset/blogs/pack_recomm/packs.csv)
 

## 2. Engineer 30- and 60-day subscriber features
HeatWave keeps feature engineering close to the data. Aggregate usage, content affinity, ARPU, and recency metrics with `user_30d_features`:

```sql
CREATE TABLE user_30d_features AS
SELECT
  s.user_id,
  s.division,
  s.age,
  s.sim_age_days,
  s.device_tier,
  s.arpu_bdt,
  SUM(du.data_mb)   AS data_mb_30d,
  SUM(du.voice_min) AS voice_min_30d,
  AVG(du.night_pct) AS avg_night_pct,
  AVG(du.yt_pct)    AS avg_yt_pct,
  AVG(du.fb_pct)    AS avg_fb_pct,
  AVG(du.wa_pct)    AS avg_wa_pct,
  COUNT(pp.pack_id) AS buys_30d,
  DATEDIFF(CURDATE(), MAX(pp.purchase_ts)) AS recency_days
FROM subscribers s
LEFT JOIN daily_usage du
       ON du.user_id = s.user_id
      AND du.usage_date >= CURDATE() - INTERVAL 30 DAY
LEFT JOIN pack_purchases pp
       ON pp.user_id = s.user_id
      AND pp.purchase_ts >= CURDATE() - INTERVAL 30 DAY
GROUP BY 1,2,3,4,5,6;
```

Complement that with 60-day categorical buying counts to tease out data vs. voice preference:

```sql
CREATE TABLE user_cat_affinity AS
SELECT
  pp.user_id,
  SUM(p.category='data')   AS data_buys_60d,
  SUM(p.category='voice')  AS voice_buys_60d,
  SUM(p.category='combo')  AS combo_buys_60d,
  SUM(p.category='social') AS social_buys_60d
FROM pack_purchases pp
JOIN packs p ON p.pack_id = pp.pack_id
WHERE pp.purchase_ts >= CURDATE() - INTERVAL 60 DAY
GROUP BY pp.user_id;
```

## 3. Create supervised labels from historic purchases
The model predicts whether a user will buy a pack within seven days after a given observation date. Build the dense training set by cross joining users and packs, then labeling positives from purchase facts:

```sql
CREATE TABLE train_samples AS
SELECT
  CAST(u.user_id AS CHAR(32)) AS user_id,
  CAST(p.pack_id AS CHAR(32)) AS pack_id,
  DATE(du.usage_date)         AS asof_date,
  CAST(
    EXISTS(
      SELECT 1
      FROM pack_purchases pp
      WHERE pp.user_id = u.user_id
        AND pp.pack_id = p.pack_id
        AND pp.purchase_ts >  du.usage_date
        AND pp.purchase_ts <= du.usage_date + INTERVAL 7 DAY
    ) AS UNSIGNED
  ) AS label
FROM (SELECT DISTINCT user_id FROM daily_usage) u
CROSS JOIN packs p
JOIN daily_usage du
  ON du.user_id = u.user_id
WHERE du.usage_date BETWEEN CURDATE() - INTERVAL 45 DAY
                        AND CURDATE() - INTERVAL 15 DAY;
```

To avoid an overwhelming number of negative examples, randomly downsample them to a 15% slice:

```sql
CREATE TABLE train_balanced AS
SELECT * FROM train_samples WHERE label = 1
UNION ALL
SELECT * FROM train_samples WHERE label = 0 AND RAND() < 0.15;
```

## 4. Assemble the HeatWave-ready training frame
Bring together subscriber attributes, usage aggregates, pack attributes, and category affinity into a single denormalized table for AutoML:

```sql
CREATE TABLE reco_training AS
SELECT
  t.user_id, t.pack_id, t.asof_date, t.label,
  u.division, u.age, u.sim_age_days, u.device_tier, u.arpu_bdt,
  u.data_mb_30d, u.voice_min_30d, u.avg_night_pct, u.avg_yt_pct,
  u.avg_fb_pct, u.avg_wa_pct, u.buys_30d, u.recency_days,
  COALESCE(a.data_buys_60d,0)   AS data_buys_60d,
  COALESCE(a.voice_buys_60d,0)  AS voice_buys_60d,
  COALESCE(a.combo_buys_60d,0)  AS combo_buys_60d,
  COALESCE(a.social_buys_60d,0) AS social_buys_60d,
  p.operator, p.category, p.data_mb, p.minutes, p.validity_d, p.price_bdt
FROM train_balanced t
LEFT JOIN user_30d_features u ON u.user_id = t.user_id
LEFT JOIN user_cat_affinity a ON a.user_id = t.user_id
LEFT JOIN packs p             ON p.pack_id  = t.pack_id;
```

## 5. Train the model with HeatWave AutoML
HeatWave AutoML optimizes model type and hyperparameters inside the database. Target the `label` column and prioritize F1 so precision/recall trade-offs stay balanced for prepaid churn mitigation:

```sql
CALL sys.ML_TRAIN(
  'telco_reco.reco_training',
  'label',
  JSON_OBJECT('task','classification','optimization_metric','f1'),
  @reco_model
);
```

The stored procedure returns a model handle (stored in `@reco_model`) that you can persist in a metadata table or application configuration.

## 6. Score daily recommendation candidates
Every morning, roll forward a candidate universe by pairing each active user with the full pack catalog and injecting the latest features:

```sql
CREATE TABLE candidates_today AS
SELECT
  u.user_id, p.pack_id,
  CURDATE() AS asof_date,
  u.division, u.age, u.sim_age_days, u.device_tier, u.arpu_bdt,
  u.data_mb_30d, u.voice_min_30d, u.avg_night_pct, u.avg_yt_pct, u.avg_fb_pct, u.avg_wa_pct,
  u.buys_30d, u.recency_days,
  COALESCE(a.data_buys_60d,0)   AS data_buys_60d,
  COALESCE(a.voice_buys_60d,0)  AS voice_buys_60d,
  COALESCE(a.combo_buys_60d,0)  AS combo_buys_60d,
  COALESCE(a.social_buys_60d,0) AS social_buys_60d,
  p.operator, p.category, p.data_mb, p.minutes, p.validity_d, p.price_bdt
FROM user_30d_features u
LEFT JOIN user_cat_affinity a ON a.user_id = u.user_id
CROSS JOIN packs p;
```

Load the trained model into memory and score the candidate table directly:

```sql
CALL sys.ML_MODEL_LOAD(@reco_model, NULL);
CALL sys.ML_PREDICT_TABLE(
  'telco_reco.candidates_today',
  @reco_model,
  'telco_reco.reco_scored',
  NULL
);
```

Rank by the probability of purchase (`p_buy`) and persist a top-three list per user:

```sql
CREATE OR REPLACE VIEW reco_ranked AS
SELECT
  user_id,
  pack_id,
  CAST(JSON_EXTRACT(ml_results, '$.probabilities.\"1\"') AS DECIMAL(10,6)) AS p_buy,
  ROW_NUMBER() OVER (
    PARTITION BY user_id
    ORDER BY CAST(JSON_EXTRACT(ml_results, '$.probabilities.\"1\"') AS DECIMAL(10,6)) DESC
  ) AS rn
FROM telco_reco.reco_scored;

CREATE TABLE recommendations_top3 AS
SELECT user_id, pack_id, p_buy
FROM reco_ranked
WHERE rn <= 3;
```

## 7. Explain recommendations with HeatWave's in-database LLM
Leverage `sys.ML_GENERATE` to draft friendly rationale strings (English, Bangla, or mixed) for each recommendation. Provide richer marketing copy by pulling pack metadata and behavioral signals into the prompt:

```sql
ALTER TABLE recommendations_top3 ADD COLUMN expl_en TEXT;

UPDATE recommendations_top3 r
JOIN packs p USING (pack_id)
JOIN user_30d_features u USING (user_id)
SET r.expl_en = sys.ML_GENERATE(
  JSON_OBJECT(
    'prompt', CONCAT(
      'User: ', u.division, ', ARPU=', ROUND(u.arpu_bdt), ' BDT; ',
      '30 Days Data=', u.data_mb_30d, 'MB, Voice=', u.voice_min_30d, ' Min. ',
      'Pack: ', p.slug, ' (', p.category, ', ', p.data_mb, 'MB, ', p.minutes, ' Min, ',
      p.validity_d, ' Day, ', p.price_bdt, ' BDT). ',
      'Write a 1-2 line reason for recommending this pack.'
    )
  ),
  JSON_OBJECT(
    'task','generation',
    'model_id','llama3.2-3b-instruct-v1',
    'language','en',
    'service_region','ap-mumbai-1'
  )
);
```

If you see the DBeaver error `ML003189 … Invalid OCI region (phx) provided`, set `service_region` to the HeatWave GenAI region where your tenancy is provisioned (e.g., `ap-mumbai-1` for South Asia). A dry-run call like `SELECT sys.ML_GENERATE(JSON_OBJECT('prompt','Test'), …);` helps confirm GenAI connectivity before running the bulk update.

## 8. Serve packs to frontline channels
Use MSISDN lookups to surface the ranked packs and explainers to CRM, app, or IVR channels:

```sql
SELECT r.user_id, p.slug, p.operator, p.category,
       p.data_mb, p.minutes, p.validity_d, p.price_bdt,
       r.p_buy, r.expl_en
FROM recommendations_top3 r
JOIN subscribers s ON s.user_id = r.user_id
JOIN packs p USING (pack_id)
WHERE s.msisdn = '880100000391';
```

With the entire life cycle — feature engineering, model training, scoring, and explanation — running inside MySQL HeatWave, BD telcos can ship personalized pack recommendations without exporting customer data or stitching together multiple services.
