---
name: 01 - Data Hygiene
assetId: af3b2213-4d58-40d5-8735-e6312b7cc8a0
type: page
---

---
title: Data Hygiene
---

# Data Hygiene

Date quality, recording lag distribution, dollar amount coverage, and pipeline freshness.

---

## Date Anomalies

{% table data="audit_base_data_quality" /%}

---

## Recording Lag by Signal Category

The gap between execution and recording varies by instrument type. Extreme outliers (>365 days) warrant investigation.

{% horizontal_bar_chart
    data="audit_recording_lag_outliers"
    x="subcategory"
    y="max(median_days_to_record)"
    title="Median Recording Lag by Subcategory"
    y_axis_options={ title="Days" }
/%}

{% table data="audit_recording_lag_outliers" %}
  {% dimension value="category" /%}
  {% dimension value="subcategory" /%}
  {% measure value="sum(total_count)" fmt="num0" /%}
  {% measure value="max(avg_days_to_record)" fmt="num0" /%}
  {% measure value="max(median_days_to_record)" fmt="num0" /%}
  {% measure value="max(p95_days_to_record)" fmt="num0" /%}
  {% measure value="max(over_1yr_lag_count)" fmt="num0" /%}
{% /table %}

---

## Amount Coverage

Dollar values drive opportunity sizing. Categories with low amount coverage produce unreliable dollar estimates.

{% horizontal_bar_chart
    data="audit_amount_data_quality"
    x="subcategory"
    y="max(pct_with_amount)"
    title="% of Instruments With Dollar Amount"
    y_axis_options={ title="%" }
/%}

{% table data="audit_amount_data_quality" %}
  {% dimension value="category" /%}
  {% dimension value="subcategory" /%}
  {% measure value="sum(total_count)" fmt="num0" /%}
  {% measure value="sum(has_any_amount_count)" fmt="num0" /%}
  {% measure value="max(pct_with_amount)" fmt="num1" /%}
{% /table %}

---

## Pipeline Freshness

{% row %}
{% big_value data="audit_data_freshness" value="max(most_recent_record)" title="Most Recent Record" /%}
{% big_value data="audit_data_freshness" value="max(days_since_last_record)" title="Days Since Last Record" /%}
{% big_value data="audit_data_freshness" value="max(records_last_7_days)" title="Last 7 Days" fmt="num0" /%}
{% big_value data="audit_data_freshness" value="max(records_last_30_days)" title="Last 30 Days" fmt="num0" /%}
{% /row %}

If `days_since_last_record` exceeds 7, investigate the ingestion pipeline.