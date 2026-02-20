---
name: 02 - Coverage
assetId: 5361d2e5-7e7f-4f7d-b363-ddf7da16da79
type: page
---

---
title: Coverage & Enrichment
---

# Coverage & Enrichment

Signal classification completeness, bridge relationship coverage, geographic assignment, tax assessment linkage, and enrichment quality.

---

## Signal Classification

Every instrument type needs a mapping in `dim.signal_category` to participate in signal analysis. Unmapped instruments are invisible to the platform.

{% table data="audit_signal_category_coverage" %}
  {% dimension value="mapping_status" /%}
  {% measure value="sum(instrument_count)" fmt="num0" /%}
  {% measure value="count(*) as type_combinations" /%}
{% /table %}

```sql unmapped_detail
SELECT instrument_type, instrument_sub_type, instrument_count
FROM audit_signal_category_coverage
WHERE mapping_status = 'UNMAPPED'
ORDER BY instrument_count DESC
```

{% table data="unmapped_detail" limit=15 /%}

---

## Mortgage Term Coverage

Mortgages without term assumptions configured are excluded from maturity analysis and refi window calculations.

{% bar_chart
    data="audit_mortgage_term_coverage"
    x="mortgage_type_group"
    y="sum(mortgage_count)"
    series="term_assumption_status"
    title="Mortgage Term Assumption Coverage"
/%}

{% table data="audit_mortgage_term_coverage" %}
  {% dimension value="term_assumption_status" /%}
  {% dimension value="mortgage_type_group" /%}
  {% dimension value="mortgage_trusted_type" /%}
  {% measure value="sum(mortgage_count)" fmt="num0" /%}
  {% measure value="sum(total_principal)" fmt="usd0" /%}
{% /table %}

---

## Bridge Relationship Coverage

{% row %}
{% big_value data="audit_bridge_relationship_coverage" value="max(total_instruments)" title="Total Instruments" fmt="num0" /%}
{% big_value data="audit_bridge_relationship_coverage" value="max(pct_with_legal_unit)" title="% With Property Link" fmt="num1" /%}
{% big_value data="audit_bridge_relationship_coverage" value="max(pct_with_party_role)" title="% With Party Link" fmt="num1" /%}
{% /row %}

---

## Geography & Tax Assessment

{% row %}
{% big_value data="audit_geography_coverage" value="max(pct_with_municipality)" title="% With Municipality" fmt="num1" /%}
{% big_value data="audit_geography_coverage" value="max(pct_with_land_use)" title="% With Land Use Code" fmt="num1" /%}
{% big_value data="audit_tax_assessment_coverage" value="max(pct_with_tax_assessment)" title="% With Tax Assessment" fmt="num1" /%}
{% /row %}

---

## Address Data Quality

{% table data="audit_address_data_quality" /%}

---

## Instrument Metadata Coverage

{% table data="audit_instrument_metadata_coverage" /%}

---

## Mortgage Eligibility Flags

{% table data="audit_mortgage_eligibility_flags" /%}

---

## Rate Data Freshness

{% table data="audit_rate_data_freshness" /%}