---
name: Home
assetId: c18a30f5-1d9f-4e71-91d9-e5161bcc6957
type: page
---

---
title: RealFACTS Signals Intelligence Platform
---

# RealFACTS Signals Intelligence Platform

Property-level signal intelligence derived from Guam Department of Land Management recorded instruments. Every mortgage, lien, deed transfer, and court filing tells a story about what's happening at a property -- and what the owner might need next.

---

## Platform Pulse

{% row %}
{% big_value data="audit_data_freshness" value="max(days_since_last_record)" title="Days Since Last Record" /%}
{% big_value data="audit_data_freshness" value="max(records_last_30_days)" title="Records (30d)" fmt="num0" /%}
{% big_value data="audit_data_freshness" value="max(records_last_90_days)" title="Records (90d)" fmt="num0" /%}
{% /row %}

---

## Signal Activity: Current vs Prior 30 Days

{% row %}
{% big_value data="gold_executive_signals_summary" value="max(signals_last_30_days)" title="Signals (Current 30d)" fmt="num0" /%}
{% big_value data="gold_executive_signals_summary" value="max(signals_prior_30_days)" title="Signals (Prior 30d)" fmt="num0" /%}
{% big_value data="gold_executive_signals_summary" value="max(owner_pressure_last_30d)" title="Owner Pressure (30d)" fmt="num0" /%}
{% big_value data="gold_executive_signals_summary" value="max(capital_leverage_volume_last_30d)" title="Capital Volume (30d)" fmt="usd0" /%}
{% /row %}

---

## Signal Classification Coverage

{% table data="audit_signal_category_coverage" %}
  {% dimension value="mapping_status" /%}
  {% measure value="sum(instrument_count)" fmt="num0" /%}
  {% measure value="count(*) as type_combinations" /%}
{% /table %}

The platform classifies 98.8% of all recorded instruments into actionable signal families. The remaining 1.2% are instrument types awaiting business logic assignment.

---

## Navigate

- [Data Quality (Audit)](/audit) -- Is the data trustworthy?
- [Signal Intelligence](/signals) -- Executive signal trends
- [Rate Spread Refi](/signals/rate-spread) -- Mortgages with rate gaps
- [Distress & Pressure](/signals/distress) -- Tax liens, judgments, lis pendens
- [Equity Trends](/signals/equity) -- Rising property values on VA/FHA mortgages
- [Alienation & Refi](/signals/alienation-refi) -- Active mortgages with recent deed transfers
- [Commercial Lending](/signals/commercial) -- Multi-property mortgage holders
- [Maturity & Retention](/signals/maturity) -- Mortgages approaching term
- [Operations](/operations/priority-list) -- Cross-signal priority list and property 360