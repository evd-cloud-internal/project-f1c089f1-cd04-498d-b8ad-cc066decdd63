---
name: 00 - Data Quality
assetId: 5fcaaab1-5888-448a-8c00-9a21aafd92c2
type: page
---

---
title: Data Quality
---

# Data Quality

Before any signal reaches a work queue, it passes through 18 audit views that answer one question: **is the data right?**

---

## Data Completeness

{% row %}
{% big_value data="audit_base_data_quality" value="max(total_records)" title="Total Instruments" fmt="num0" /%}
{% big_value data="audit_base_data_quality" value="max(pct_null_date_recorded)" title="% Missing Record Date" fmt="num1" /%}
{% big_value data="audit_base_data_quality" value="max(pct_null_date_executed)" title="% Missing Execution Date" fmt="num1" /%}
{% /row %}

---

## Bridge & Join Coverage

Every instrument needs connections: to a property (legal unit), to the people involved (parties), and to a location (municipality).

{% row %}
{% big_value data="audit_bridge_relationship_coverage" value="max(pct_with_legal_unit)" title="% Linked to Property" fmt="num1" /%}
{% big_value data="audit_bridge_relationship_coverage" value="max(pct_with_party_role)" title="% Linked to Party" fmt="num1" /%}
{% big_value data="audit_geography_coverage" value="max(pct_with_municipality)" title="% With Municipality" fmt="num1" /%}
{% big_value data="audit_tax_assessment_coverage" value="max(pct_with_tax_assessment)" title="% With Tax Assessment" fmt="num1" /%}
{% /row %}

---

## Fan-Out Health

Fan-out monitoring catches anomalous join multiplication. A single instrument linked to 2,000+ properties is a blanket instrument -- legitimate but requiring special handling.

{% table data="audit_fan_out_monitoring" %}
  {% dimension value="fan_type" /%}
  {% measure value="max(total_instruments)" fmt="num0" /%}
  {% measure value="max(avg_fan_out)" fmt="num1" /%}
  {% measure value="max(max_fan_out)" fmt="num0" /%}
  {% measure value="max(over_10_count)" fmt="num0" /%}
  {% measure value="max(p99_fan_out)" fmt="num0" /%}
{% /table %}

---

## Drill-Down Pages

- [Data Hygiene](/audit/data-hygiene) -- Date anomalies, recording lag, amount coverage, pipeline freshness
- [Coverage & Enrichment](/audit/coverage) -- Signal classification, bridge relationships, geography, tax assessments
- [Trends](/audit/trends) -- Monthly signal volume by category, vintage distribution
- [Entity Quality](/audit/entity-quality) -- Party contact coverage, snapshot resolution health