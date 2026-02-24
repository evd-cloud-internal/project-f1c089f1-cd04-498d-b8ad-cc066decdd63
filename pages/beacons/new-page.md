---
name: New Page
assetId: e652b2c0-dedb-4a3c-a22e-1e2ceddbd76d
type: page
---

---
title: Transition Flows
description: Instrument lifecycle transitions by property and party linkage
---

# Transition Flows

What happens next on a property or in a person's recording history? These views trace every recorded instrument to the one that follows it, partitioned by property (legal_unit_id) or party (party_id). The result is a complete map of how Guam's recorded instruments flow from one type to another--and how long each transition takes.

Three levels of detail: signal categories (6 nodes), signal subcategories (15 nodes), and individual instrument types (108+ nodes).

---

## Level 1: Category Flows

Category-to-category transitions show the broadest patterns. Each node is one of the six signal categories from dim.signal_category.

### Property Lifecycle

```sql property_category_flows
SELECT
    source_category,
    target_category,
    SUM(transition_count) AS transition_count
FROM silver_instrument_transitions_property
GROUP BY source_category, target_category
ORDER BY transition_count DESC
```

{% sankey_chart
    data="property_category_flows"
    source="source_category"
    target="target_category"
    value="transition_count"
    title="Property: Category-to-Category Flows (938K transitions)"
/%}

```sql property_category_matrix
SELECT
    source_category AS "Source Category",
    target_category AS "Target Category",
    SUM(transition_count) AS "Transitions",
    ROUND(AVG(median_days_between), 0) AS "Median Days",
    ROUND(AVG(avg_days_between), 0) AS "Avg Days"
FROM silver_instrument_transitions_property
GROUP BY source_category, target_category
ORDER BY SUM(transition_count) DESC
```

{% table data="property_category_matrix" /%}


### Party Lifecycle

```sql party_category_flows
SELECT
    source_category,
    target_category,
    SUM(transition_count) AS transition_count
FROM silver_instrument_transitions_party
GROUP BY source_category, target_category
ORDER BY transition_count DESC
```

{% sankey_chart
    data="party_category_flows"
    source="source_category"
    target="target_category"
    value="transition_count"
    title="Party: Category-to-Category Flows (1.78M transitions)"
/%}

```sql party_category_matrix
SELECT
    source_category AS "Source Category",
    target_category AS "Target Category",
    SUM(transition_count) AS "Transitions",
    ROUND(AVG(median_days_between), 0) AS "Median Days",
    ROUND(AVG(avg_days_between), 0) AS "Avg Days"
FROM silver_instrument_transitions_party
GROUP BY source_category, target_category
ORDER BY SUM(transition_count) DESC
```

{% table data="party_category_matrix" /%}


### Combined (Property + Party)

```sql combined_category_flows
SELECT
    source_category,
    target_category,
    SUM(transition_count) AS transition_count
FROM silver_instrument_transitions_combined
GROUP BY source_category, target_category
ORDER BY transition_count DESC
```

{% sankey_chart
    data="combined_category_flows"
    source="source_category"
    target="target_category"
    value="transition_count"
    title="Combined: Category-to-Category Flows (2.72M transitions)"
/%}

---

## Level 2: Subcategory Flows

Subcategory-to-subcategory transitions reveal the signal families driving lifecycle movement. Each node is one of the 15 signal subcategories.

### Property Lifecycle

```sql property_subcategory_flows
SELECT
    source_subcategory,
    target_subcategory,
    SUM(transition_count) AS transition_count
FROM silver_instrument_transitions_property
GROUP BY source_subcategory, target_subcategory
ORDER BY transition_count DESC
```

{% sankey_chart
    data="property_subcategory_flows"
    source="source_subcategory"
    target="target_subcategory"
    value="transition_count"
    title="Property: Subcategory-to-Subcategory Flows"
/%}

```sql property_subcategory_matrix
SELECT
    source_subcategory AS "Source Subcategory",
    target_subcategory AS "Target Subcategory",
    SUM(transition_count) AS "Transitions",
    ROUND(AVG(median_days_between), 0) AS "Median Days",
    ROUND(AVG(avg_days_between), 0) AS "Avg Days"
FROM silver_instrument_transitions_property
GROUP BY source_subcategory, target_subcategory
ORDER BY SUM(transition_count) DESC
LIMIT 30
```

{% table data="property_subcategory_matrix" /%}


### Party Lifecycle

```sql party_subcategory_flows
SELECT
    source_subcategory,
    target_subcategory,
    SUM(transition_count) AS transition_count
FROM silver_instrument_transitions_party
GROUP BY source_subcategory, target_subcategory
ORDER BY transition_count DESC
```

{% sankey_chart
    data="party_subcategory_flows"
    source="source_subcategory"
    target="target_subcategory"
    value="transition_count"
    title="Party: Subcategory-to-Subcategory Flows"
/%}

```sql party_subcategory_matrix
SELECT
    source_subcategory AS "Source Subcategory",
    target_subcategory AS "Target Subcategory",
    SUM(transition_count) AS "Transitions",
    ROUND(AVG(median_days_between), 0) AS "Median Days",
    ROUND(AVG(avg_days_between), 0) AS "Avg Days"
FROM silver_instrument_transitions_party
GROUP BY source_subcategory, target_subcategory
ORDER BY SUM(transition_count) DESC
LIMIT 30
```

{% table data="party_subcategory_matrix" /%}


### Combined (Property + Party)

```sql combined_subcategory_flows
SELECT
    source_subcategory,
    target_subcategory,
    SUM(transition_count) AS transition_count
FROM silver_instrument_transitions_combined
GROUP BY source_subcategory, target_subcategory
ORDER BY transition_count DESC
```

{% sankey_chart
    data="combined_subcategory_flows"
    source="source_subcategory"
    target="target_subcategory"
    value="transition_count"
    title="Combined: Subcategory-to-Subcategory Flows"
/%}

---

## Level 3: Instrument-Level Transitions by Subcategory

Each section below isolates one signal subcategory and shows every instrument-to-instrument transition originating from that family. Heatmaps show volume intensity; tables show timing. Property linkage (what happens next on the same parcel) is the primary view. Sections ordered by total transition volume.

---

### Transaction Timing Triggers

Instruments signaling ownership transfers, sales, and conveyances--the core transactional lifecycle on a property.

```sql transaction_timing_property
SELECT
    source_display_name,
    target_display_name,
    SUM(transition_count) AS transition_count,
    ROUND(AVG(median_days_between), 0) AS median_days
FROM silver_instrument_transitions_property
WHERE source_subcategory = 'Transaction Timing Triggers'
GROUP BY source_display_name, target_display_name
ORDER BY transition_count DESC
```

{% heatmap
    data="transaction_timing_property"
    x="source_display_name"
    y="target_display_name"
    value="transition_count"
    title="Transaction Timing Triggers: Property Transitions"
/%}

```sql transaction_timing_property_top
SELECT
    source_display_name AS "Source",
    target_display_name AS "Target",
    SUM(transition_count) AS "Transitions",
    ROUND(AVG(median_days_between), 0) AS "Median Days",
    ROUND(AVG(avg_days_between), 0) AS "Avg Days"
FROM silver_instrument_transitions_property
WHERE source_subcategory = 'Transaction Timing Triggers'
GROUP BY source_display_name, target_display_name
ORDER BY SUM(transition_count) DESC
LIMIT 25
```

{% table data="transaction_timing_property_top" /%}

```sql transaction_timing_party
SELECT
    source_display_name,
    target_display_name,
    SUM(transition_count) AS transition_count,
    ROUND(AVG(median_days_between), 0) AS median_days
FROM silver_instrument_transitions_party
WHERE source_subcategory = 'Transaction Timing Triggers'
GROUP BY source_display_name, target_display_name
ORDER BY transition_count DESC
```

{% heatmap
    data="transaction_timing_party"
    x="source_display_name"
    y="target_display_name"
    value="transition_count"
    title="Transaction Timing Triggers: Party Transitions"
/%}

---

### Capital & Leverage Intelligence

Debt structure instruments: mortgages, modifications, subordinations, promissory notes. Borrowers restructuring debt are active lending targets.

```sql capital_leverage_property
SELECT
    source_display_name,
    target_display_name,
    SUM(transition_count) AS transition_count,
    ROUND(AVG(median_days_between), 0) AS median_days
FROM silver_instrument_transitions_property
WHERE source_subcategory = 'Capital & Leverage Intelligence'
GROUP BY source_display_name, target_display_name
ORDER BY transition_count DESC
```

{% heatmap
    data="capital_leverage_property"
    x="source_display_name"
    y="target_display_name"
    value="transition_count"
    title="Capital & Leverage: Property Transitions"
/%}

```sql capital_leverage_property_top
SELECT
    source_display_name AS "Source",
    target_display_name AS "Target",
    SUM(transition_count) AS "Transitions",
    ROUND(AVG(median_days_between), 0) AS "Median Days",
    ROUND(AVG(avg_days_between), 0) AS "Avg Days"
FROM silver_instrument_transitions_property
WHERE source_subcategory = 'Capital & Leverage Intelligence'
GROUP BY source_display_name, target_display_name
ORDER BY SUM(transition_count) DESC
LIMIT 25
```

{% table data="capital_leverage_property_top" /%}

```sql capital_leverage_party
SELECT
    source_display_name,
    target_display_name,
    SUM(transition_count) AS transition_count,
    ROUND(AVG(median_days_between), 0) AS median_days
FROM silver_instrument_transitions_party
WHERE source_subcategory = 'Capital & Leverage Intelligence'
GROUP BY source_display_name, target_display_name
ORDER BY transition_count DESC
```

{% heatmap
    data="capital_leverage_party"
    x="source_display_name"
    y="target_display_name"
    value="transition_count"
    title="Capital & Leverage: Party Transitions"
/%}

---

### Title and Ownership Complexity

Instruments affecting title clarity: affidavits, amendments, powers of attorney, trust documents.

```sql title_ownership_property
SELECT
    source_display_name,
    target_display_name,
    SUM(transition_count) AS transition_count,
    ROUND(AVG(median_days_between), 0) AS median_days
FROM silver_instrument_transitions_property
WHERE source_subcategory = 'Title and Ownership Complexity'
GROUP BY source_display_name, target_display_name
ORDER BY transition_count DESC
```

{% heatmap
    data="title_ownership_property"
    x="source_display_name"
    y="target_display_name"
    value="transition_count"
    title="Title and Ownership Complexity: Property Transitions"
/%}

```sql title_ownership_property_top
SELECT
    source_display_name AS "Source",
    target_display_name AS "Target",
    SUM(transition_count) AS "Transitions",
    ROUND(AVG(median_days_between), 0) AS "Median Days",
    ROUND(AVG(avg_days_between), 0) AS "Avg Days"
FROM silver_instrument_transitions_property
WHERE source_subcategory = 'Title and Ownership Complexity'
GROUP BY source_display_name, target_display_name
ORDER BY SUM(transition_count) DESC
LIMIT 25
```

{% table data="title_ownership_property_top" /%}

```sql title_ownership_party
SELECT
    source_display_name,
    target_display_name,
    SUM(transition_count) AS transition_count,
    ROUND(AVG(median_days_between), 0) AS median_days
FROM silver_instrument_transitions_party
WHERE source_subcategory = 'Title and Ownership Complexity'
GROUP BY source_display_name, target_display_name
ORDER BY transition_count DESC
```

{% heatmap
    data="title_ownership_party"
    x="source_display_name"
    y="target_display_name"
    value="transition_count"
    title="Title and Ownership Complexity: Party Transitions"
/%}

---

### Equity Unlock Signals

Release and satisfaction instruments: releases of mortgage, satisfactions, discharges. Signals a lien clearing--often followed by new lending.

```sql equity_unlock_property
SELECT
    source_display_name,
    target_display_name,
    SUM(transition_count) AS transition_count,
    ROUND(AVG(median_days_between), 0) AS median_days
FROM silver_instrument_transitions_property
WHERE source_subcategory = 'Equity Unlock Signals'
GROUP BY source_display_name, target_display_name
ORDER BY transition_count DESC
```

{% heatmap
    data="equity_unlock_property"
    x="source_display_name"
    y="target_display_name"
    value="transition_count"
    title="Equity Unlock Signals: Property Transitions"
/%}

```sql equity_unlock_property_top
SELECT
    source_display_name AS "Source",
    target_display_name AS "Target",
    SUM(transition_count) AS "Transitions",
    ROUND(AVG(median_days_between), 0) AS "Median Days",
    ROUND(AVG(avg_days_between), 0) AS "Avg Days"
FROM silver_instrument_transitions_property
WHERE source_subcategory = 'Equity Unlock Signals'
GROUP BY source_display_name, target_display_name
ORDER BY SUM(transition_count) DESC
LIMIT 25
```

{% table data="equity_unlock_property_top" /%}

```sql equity_unlock_party
SELECT
    source_display_name,
    target_display_name,
    SUM(transition_count) AS transition_count,
    ROUND(AVG(median_days_between), 0) AS median_days
FROM silver_instrument_transitions_party
WHERE source_subcategory = 'Equity Unlock Signals'
GROUP BY source_display_name, target_display_name
ORDER BY transition_count DESC
```

{% heatmap
    data="equity_unlock_party"
    x="source_display_name"
    y="target_display_name"
    value="transition_count"
    title="Equity Unlock Signals: Party Transitions"
/%}

---

### Life and Legal Triggers

Probate, divorce, death filings, guardianship, distribution orders. Life events that force or enable property transactions.

```sql life_legal_property
SELECT
    source_display_name,
    target_display_name,
    SUM(transition_count) AS transition_count,
    ROUND(AVG(median_days_between), 0) AS median_days
FROM silver_instrument_transitions_property
WHERE source_subcategory = 'Life and Legal Triggers'
GROUP BY source_display_name, target_display_name
ORDER BY transition_count DESC
```

{% heatmap
    data="life_legal_property"
    x="source_display_name"
    y="target_display_name"
    value="transition_count"
    title="Life and Legal Triggers: Property Transitions"
/%}

```sql life_legal_property_top
SELECT
    source_display_name AS "Source",
    target_display_name AS "Target",
    SUM(transition_count) AS "Transitions",
    ROUND(AVG(median_days_between), 0) AS "Median Days",
    ROUND(AVG(avg_days_between), 0) AS "Avg Days"
FROM silver_instrument_transitions_property
WHERE source_subcategory = 'Life and Legal Triggers'
GROUP BY source_display_name, target_display_name
ORDER BY SUM(transition_count) DESC
LIMIT 25
```

{% table data="life_legal_property_top" /%}

```sql life_legal_party
SELECT
    source_display_name,
    target_display_name,
    SUM(transition_count) AS transition_count,
    ROUND(AVG(median_days_between), 0) AS median_days
FROM silver_instrument_transitions_party
WHERE source_subcategory = 'Life and Legal Triggers'
GROUP BY source_display_name, target_display_name
ORDER BY transition_count DESC
```

{% heatmap
    data="life_legal_party"
    x="source_display_name"
    y="target_display_name"
    value="transition_count"
    title="Life and Legal Triggers: Party Transitions"
/%}

---

### Title Registry

Certificate of Title recordings--the foundation document linking instruments to legal parcels.

```sql title_registry_property
SELECT
    source_display_name,
    target_display_name,
    SUM(transition_count) AS transition_count,
    ROUND(AVG(median_days_between), 0) AS median_days
FROM silver_instrument_transitions_property
WHERE source_subcategory = 'Title Registry'
GROUP BY source_display_name, target_display_name
ORDER BY transition_count DESC
```

{% heatmap
    data="title_registry_property"
    x="source_display_name"
    y="target_display_name"
    value="transition_count"
    title="Title Registry: Property Transitions"
/%}

```sql title_registry_property_top
SELECT
    source_display_name AS "Source",
    target_display_name AS "Target",
    SUM(transition_count) AS "Transitions",
    ROUND(AVG(median_days_between), 0) AS "Median Days",
    ROUND(AVG(avg_days_between), 0) AS "Avg Days"
FROM silver_instrument_transitions_property
WHERE source_subcategory = 'Title Registry'
GROUP BY source_display_name, target_display_name
ORDER BY SUM(transition_count) DESC
LIMIT 25
```

{% table data="title_registry_property_top" /%}

```sql title_registry_party
SELECT
    source_display_name,
    target_display_name,
    SUM(transition_count) AS transition_count,
    ROUND(AVG(median_days_between), 0) AS median_days
FROM silver_instrument_transitions_party
WHERE source_subcategory = 'Title Registry'
GROUP BY source_display_name, target_display_name
ORDER BY transition_count DESC
```

{% heatmap
    data="title_registry_party"
    x="source_display_name"
    y="target_display_name"
    value="transition_count"
    title="Title Registry: Party Transitions"
/%}

---

### Deal Complexity Events

Assignment and lease instruments that add complexity to the deal chain.

```sql deal_complexity_property
SELECT
    source_display_name,
    target_display_name,
    SUM(transition_count) AS transition_count,
    ROUND(AVG(median_days_between), 0) AS median_days
FROM silver_instrument_transitions_property
WHERE source_subcategory = 'Deal Complexity Events'
GROUP BY source_display_name, target_display_name
ORDER BY transition_count DESC
```

{% heatmap
    data="deal_complexity_property"
    x="source_display_name"
    y="target_display_name"
    value="transition_count"
    title="Deal Complexity Events: Property Transitions"
/%}

```sql deal_complexity_property_top
SELECT
    source_display_name AS "Source",
    target_display_name AS "Target",
    SUM(transition_count) AS "Transitions",
    ROUND(AVG(median_days_between), 0) AS "Median Days",
    ROUND(AVG(avg_days_between), 0) AS "Avg Days"
FROM silver_instrument_transitions_property
WHERE source_subcategory = 'Deal Complexity Events'
GROUP BY source_display_name, target_display_name
ORDER BY SUM(transition_count) DESC
LIMIT 25
```

{% table data="deal_complexity_property_top" /%}

```sql deal_complexity_party
SELECT
    source_display_name,
    target_display_name,
    SUM(transition_count) AS transition_count,
    ROUND(AVG(median_days_between), 0) AS median_days
FROM silver_instrument_transitions_party
WHERE source_subcategory = 'Deal Complexity Events'
GROUP BY source_display_name, target_display_name
ORDER BY transition_count DESC
```

{% heatmap
    data="deal_complexity_party"
    x="source_display_name"
    y="target_display_name"
    value="transition_count"
    title="Deal Complexity Events: Party Transitions"
/%}

---

### Forced Sale Signals

Foreclosure, power of sale, and judicial sale instruments. High-distress indicators.

```sql forced_sale_property
SELECT
    source_display_name,
    target_display_name,
    SUM(transition_count) AS transition_count,
    ROUND(AVG(median_days_between), 0) AS median_days
FROM silver_instrument_transitions_property
WHERE source_subcategory = 'Forced Sale Signals'
GROUP BY source_display_name, target_display_name
ORDER BY transition_count DESC
```

{% heatmap
    data="forced_sale_property"
    x="source_display_name"
    y="target_display_name"
    value="transition_count"
    title="Forced Sale Signals: Property Transitions"
/%}

```sql forced_sale_property_top
SELECT
    source_display_name AS "Source",
    target_display_name AS "Target",
    SUM(transition_count) AS "Transitions",
    ROUND(AVG(median_days_between), 0) AS "Median Days",
    ROUND(AVG(avg_days_between), 0) AS "Avg Days"
FROM silver_instrument_transitions_property
WHERE source_subcategory = 'Forced Sale Signals'
GROUP BY source_display_name, target_display_name
ORDER BY SUM(transition_count) DESC
LIMIT 25
```

{% table data="forced_sale_property_top" /%}

```sql forced_sale_party
SELECT
    source_display_name,
    target_display_name,
    SUM(transition_count) AS transition_count,
    ROUND(AVG(median_days_between), 0) AS median_days
FROM silver_instrument_transitions_party
WHERE source_subcategory = 'Forced Sale Signals'
GROUP BY source_display_name, target_display_name
ORDER BY transition_count DESC
```

{% heatmap
    data="forced_sale_party"
    x="source_display_name"
    y="target_display_name"
    value="transition_count"
    title="Forced Sale Signals: Party Transitions"
/%}

---

### Owner Pressure Signals

Tax liens, judgments, notices of default, lis pendens. Signals of financial distress on property owners.

```sql owner_pressure_property
SELECT
    source_display_name,
    target_display_name,
    SUM(transition_count) AS transition_count,
    ROUND(AVG(median_days_between), 0) AS median_days
FROM silver_instrument_transitions_property
WHERE source_subcategory = 'Owner Pressure Signals'
GROUP BY source_display_name, target_display_name
ORDER BY transition_count DESC
```

{% heatmap
    data="owner_pressure_property"
    x="source_display_name"
    y="target_display_name"
    value="transition_count"
    title="Owner Pressure Signals: Property Transitions"
/%}

```sql owner_pressure_property_top
SELECT
    source_display_name AS "Source",
    target_display_name AS "Target",
    SUM(transition_count) AS "Transitions",
    ROUND(AVG(median_days_between), 0) AS "Median Days",
    ROUND(AVG(avg_days_between), 0) AS "Avg Days"
FROM silver_instrument_transitions_property
WHERE source_subcategory = 'Owner Pressure Signals'
GROUP BY source_display_name, target_display_name
ORDER BY SUM(transition_count) DESC
LIMIT 25
```

{% table data="owner_pressure_property_top" /%}

```sql owner_pressure_party
SELECT
    source_display_name,
    target_display_name,
    SUM(transition_count) AS transition_count,
    ROUND(AVG(median_days_between), 0) AS median_days
FROM silver_instrument_transitions_party
WHERE source_subcategory = 'Owner Pressure Signals'
GROUP BY source_display_name, target_display_name
ORDER BY transition_count DESC
```

{% heatmap
    data="owner_pressure_party"
    x="source_display_name"
    y="target_display_name"
    value="transition_count"
    title="Owner Pressure Signals: Party Transitions"
/%}

---

### Administrative

Permits, designations, assessments, and other administrative recordings.

```sql administrative_property
SELECT
    source_display_name,
    target_display_name,
    SUM(transition_count) AS transition_count,
    ROUND(AVG(median_days_between), 0) AS median_days
FROM silver_instrument_transitions_property
WHERE source_subcategory = 'Administrative'
GROUP BY source_display_name, target_display_name
ORDER BY transition_count DESC
```

{% heatmap
    data="administrative_property"
    x="source_display_name"
    y="target_display_name"
    value="transition_count"
    title="Administrative: Property Transitions"
/%}

```sql administrative_property_top
SELECT
    source_display_name AS "Source",
    target_display_name AS "Target",
    SUM(transition_count) AS "Transitions",
    ROUND(AVG(median_days_between), 0) AS "Median Days",
    ROUND(AVG(avg_days_between), 0) AS "Avg Days"
FROM silver_instrument_transitions_property
WHERE source_subcategory = 'Administrative'
GROUP BY source_display_name, target_display_name
ORDER BY SUM(transition_count) DESC
LIMIT 25
```

{% table data="administrative_property_top" /%}

```sql administrative_party
SELECT
    source_display_name,
    target_display_name,
    SUM(transition_count) AS transition_count,
    ROUND(AVG(median_days_between), 0) AS median_days
FROM silver_instrument_transitions_party
WHERE source_subcategory = 'Administrative'
GROUP BY source_display_name, target_display_name
ORDER BY transition_count DESC
```

{% heatmap
    data="administrative_party"
    x="source_display_name"
    y="target_display_name"
    value="transition_count"
    title="Administrative: Party Transitions"
/%}

---

### Blocker Clearance Signals

Cancellations, terminations, and satisfaction of liens. Instruments that clear encumbrances.

```sql blocker_clearance_property
SELECT
    source_display_name,
    target_display_name,
    SUM(transition_count) AS transition_count,
    ROUND(AVG(median_days_between), 0) AS median_days
FROM silver_instrument_transitions_property
WHERE source_subcategory = 'Blocker Clearance Signals'
GROUP BY source_display_name, target_display_name
ORDER BY transition_count DESC
```

{% heatmap
    data="blocker_clearance_property"
    x="source_display_name"
    y="target_display_name"
    value="transition_count"
    title="Blocker Clearance Signals: Property Transitions"
/%}

```sql blocker_clearance_property_top
SELECT
    source_display_name AS "Source",
    target_display_name AS "Target",
    SUM(transition_count) AS "Transitions",
    ROUND(AVG(median_days_between), 0) AS "Median Days",
    ROUND(AVG(avg_days_between), 0) AS "Avg Days"
FROM silver_instrument_transitions_property
WHERE source_subcategory = 'Blocker Clearance Signals'
GROUP BY source_display_name, target_display_name
ORDER BY SUM(transition_count) DESC
LIMIT 25
```

{% table data="blocker_clearance_property_top" /%}

```sql blocker_clearance_party
SELECT
    source_display_name,
    target_display_name,
    SUM(transition_count) AS transition_count,
    ROUND(AVG(median_days_between), 0) AS median_days
FROM silver_instrument_transitions_party
WHERE source_subcategory = 'Blocker Clearance Signals'
GROUP BY source_display_name, target_display_name
ORDER BY transition_count DESC
```

{% heatmap
    data="blocker_clearance_party"
    x="source_display_name"
    y="target_display_name"
    value="transition_count"
    title="Blocker Clearance Signals: Party Transitions"
/%}

---

### Supply Signals

Maps, subdivisions, and construction-related instruments signaling new inventory entering the market.

```sql supply_signals_property
SELECT
    source_display_name,
    target_display_name,
    SUM(transition_count) AS transition_count,
    ROUND(AVG(median_days_between), 0) AS median_days
FROM silver_instrument_transitions_property
WHERE source_subcategory = 'Supply Signals'
GROUP BY source_display_name, target_display_name
ORDER BY transition_count DESC
```

{% heatmap
    data="supply_signals_property"
    x="source_display_name"
    y="target_display_name"
    value="transition_count"
    title="Supply Signals: Property Transitions"
/%}

```sql supply_signals_property_top
SELECT
    source_display_name AS "Source",
    target_display_name AS "Target",
    SUM(transition_count) AS "Transitions",
    ROUND(AVG(median_days_between), 0) AS "Median Days",
    ROUND(AVG(avg_days_between), 0) AS "Avg Days"
FROM silver_instrument_transitions_property
WHERE source_subcategory = 'Supply Signals'
GROUP BY source_display_name, target_display_name
ORDER BY SUM(transition_count) DESC
LIMIT 25
```

{% table data="supply_signals_property_top" /%}

```sql supply_signals_party
SELECT
    source_display_name,
    target_display_name,
    SUM(transition_count) AS transition_count,
    ROUND(AVG(median_days_between), 0) AS median_days
FROM silver_instrument_transitions_party
WHERE source_subcategory = 'Supply Signals'
GROUP BY source_display_name, target_display_name
ORDER BY transition_count DESC
```

{% heatmap
    data="supply_signals_party"
    x="source_display_name"
    y="target_display_name"
    value="transition_count"
    title="Supply Signals: Party Transitions"
/%}

---

### Regulatory, Risk & Claims Intelligence

Regulatory filings, environmental liens, and risk-related instruments.

```sql regulatory_risk_property
SELECT
    source_display_name,
    target_display_name,
    SUM(transition_count) AS transition_count,
    ROUND(AVG(median_days_between), 0) AS median_days
FROM silver_instrument_transitions_property
WHERE source_subcategory = 'Regulatory, Risk & Claims Intelligence'
GROUP BY source_display_name, target_display_name
ORDER BY transition_count DESC
```

{% heatmap
    data="regulatory_risk_property"
    x="source_display_name"
    y="target_display_name"
    value="transition_count"
    title="Regulatory, Risk & Claims: Property Transitions"
/%}

```sql regulatory_risk_property_top
SELECT
    source_display_name AS "Source",
    target_display_name AS "Target",
    SUM(transition_count) AS "Transitions",
    ROUND(AVG(median_days_between), 0) AS "Median Days",
    ROUND(AVG(avg_days_between), 0) AS "Avg Days"
FROM silver_instrument_transitions_property
WHERE source_subcategory = 'Regulatory, Risk & Claims Intelligence'
GROUP BY source_display_name, target_display_name
ORDER BY SUM(transition_count) DESC
LIMIT 25
```

{% table data="regulatory_risk_property_top" /%}

```sql regulatory_risk_party
SELECT
    source_display_name,
    target_display_name,
    SUM(transition_count) AS transition_count,
    ROUND(AVG(median_days_between), 0) AS median_days
FROM silver_instrument_transitions_party
WHERE source_subcategory = 'Regulatory, Risk & Claims Intelligence'
GROUP BY source_display_name, target_display_name
ORDER BY transition_count DESC
```

{% heatmap
    data="regulatory_risk_party"
    x="source_display_name"
    y="target_display_name"
    value="transition_count"
    title="Regulatory, Risk & Claims: Party Transitions"
/%}

---

### Deal Propensity Events

Rescissions and amendments--instruments that signal active deal negotiation or modification.

```sql deal_propensity_property
SELECT
    source_display_name,
    target_display_name,
    SUM(transition_count) AS transition_count,
    ROUND(AVG(median_days_between), 0) AS median_days
FROM silver_instrument_transitions_property
WHERE source_subcategory = 'Deal Propensity Events'
GROUP BY source_display_name, target_display_name
ORDER BY transition_count DESC
```

{% heatmap
    data="deal_propensity_property"
    x="source_display_name"
    y="target_display_name"
    value="transition_count"
    title="Deal Propensity Events: Property Transitions"
/%}

```sql deal_propensity_property_top
SELECT
    source_display_name AS "Source",
    target_display_name AS "Target",
    SUM(transition_count) AS "Transitions",
    ROUND(AVG(median_days_between), 0) AS "Median Days",
    ROUND(AVG(avg_days_between), 0) AS "Avg Days"
FROM silver_instrument_transitions_property
WHERE source_subcategory = 'Deal Propensity Events'
GROUP BY source_display_name, target_display_name
ORDER BY SUM(transition_count) DESC
LIMIT 25
```

{% table data="deal_propensity_property_top" /%}

```sql deal_propensity_party
SELECT
    source_display_name,
    target_display_name,
    SUM(transition_count) AS transition_count,
    ROUND(AVG(median_days_between), 0) AS median_days
FROM silver_instrument_transitions_party
WHERE source_subcategory = 'Deal Propensity Events'
GROUP BY source_display_name, target_display_name
ORDER BY transition_count DESC
```

{% heatmap
    data="deal_propensity_party"
    x="source_display_name"
    y="target_display_name"
    value="transition_count"
    title="Deal Propensity Events: Party Transitions"
/%}

---

### Liens and Encumbrances

Lien instruments that create encumbrances against property. Smallest subcategory by volume.

```sql liens_encumbrances_property
SELECT
    source_display_name,
    target_display_name,
    SUM(transition_count) AS transition_count,
    ROUND(AVG(median_days_between), 0) AS median_days
FROM silver_instrument_transitions_property
WHERE source_subcategory = 'Liens and Encumbrances'
GROUP BY source_display_name, target_display_name
ORDER BY transition_count DESC
```

{% heatmap
    data="liens_encumbrances_property"
    x="source_display_name"
    y="target_display_name"
    value="transition_count"
    title="Liens and Encumbrances: Property Transitions"
/%}

```sql liens_encumbrances_property_top
SELECT
    source_display_name AS "Source",
    target_display_name AS "Target",
    SUM(transition_count) AS "Transitions",
    ROUND(AVG(median_days_between), 0) AS "Median Days",
    ROUND(AVG(avg_days_between), 0) AS "Avg Days"
FROM silver_instrument_transitions_property
WHERE source_subcategory = 'Liens and Encumbrances'
GROUP BY source_display_name, target_display_name
ORDER BY SUM(transition_count) DESC
LIMIT 25
```

{% table data="liens_encumbrances_property_top" /%}

```sql liens_encumbrances_party
SELECT
    source_display_name,
    target_display_name,
    SUM(transition_count) AS transition_count,
    ROUND(AVG(median_days_between), 0) AS median_days
FROM silver_instrument_transitions_party
WHERE source_subcategory = 'Liens and Encumbrances'
GROUP BY source_display_name, target_display_name
ORDER BY transition_count DESC
```

{% heatmap
    data="liens_encumbrances_party"
    x="source_display_name"
    y="target_display_name"
    value="transition_count"
    title="Liens and Encumbrances: Party Transitions"
/%}

---

## Appendix: Data Sources

- **Property transitions:** `silver_instrument_transitions_property` -- partitioned by legal_unit_id, LEAD() ordered by date_recorded + id
- **Party transitions:** `silver_instrument_transitions_party` -- partitioned by party_id, LEAD() ordered by date_recorded + id
- **Combined transitions:** `silver_instrument_transitions_combined` -- UNION ALL of property and party with linkage_type tag
- **Classification:** `dim.signal_category` -- 6 categories, 15 subcategories, 165 instrument type mappings
- **Display names:** `dim.dictionary` -- COALESCE(sub_type display, type display, raw code)