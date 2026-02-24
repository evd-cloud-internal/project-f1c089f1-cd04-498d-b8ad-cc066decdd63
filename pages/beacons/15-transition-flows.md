---
name: 15 - Transition Flows
assetId: e652b2c0-dedb-4a3c-a22e-1e2ceddbd76d
type: page
---

# Transition Flows

What happens next on a property or in a person's recording history? These views trace every recorded instrument to the one that follows it, partitioned by property (legal_unit_id) or party (party_id). The result is a complete map of how Guam's recorded instruments flow from one type to another--and how long each transition takes.

Three levels of detail: signal categories (6 nodes), signal subcategories (15 nodes), and individual instrument types (108+ nodes).

## Level 1: Category Flows

Category-to-category transitions show the broadest patterns. Each node is one of the six signal categories from dim.signal_category.

### Property Lifecycle

```sql l1_prop
select
  source_category,
  target_category,
  sum(transition_count) as transition_count,
  round(avg(median_days_between), 0) as median_days,
  round(avg(avg_days_between), 0) as avg_days
from silver_instrument_transitions_property
group by source_category, target_category
order by transition_count desc
```

{% sankey_chart
    data="l1_prop"
    source="source_category"
    target="target_category"
    value="sum(transition_count)"
    title="Property: Category-to-Category Flows"
/%}

{% table data="l1_prop" page_size=200 /%}

### Party Lifecycle

```sql l1_party
select
  source_category,
  target_category,
  sum(transition_count) as transition_count,
  round(avg(median_days_between), 0) as median_days,
  round(avg(avg_days_between), 0) as avg_days
from silver_instrument_transitions_party
group by source_category, target_category
order by transition_count desc
```

{% sankey_chart
    data="l1_party"
    source="source_category"
    target="target_category"
    value="sum(transition_count)"
    title="Party: Category-to-Category Flows"
/%}

{% table data="l1_party" page_size=200 /%}

### Combined (Property + Party)

```sql l1_combined
select
  source_category,
  target_category,
  sum(transition_count) as transition_count,
  round(avg(median_days_between), 0) as median_days,
  round(avg(avg_days_between), 0) as avg_days
from silver_instrument_transitions_combined
group by source_category, target_category
order by transition_count desc
```

{% sankey_chart
    data="l1_combined"
    source="source_category"
    target="target_category"
    value="sum(transition_count)"
    title="Combined: Category-to-Category Flows"
/%}

{% table data="l1_combined" page_size=200 /%}

## Level 2: Subcategory Flows

Subcategory-to-subcategory transitions reveal the signal families driving lifecycle movement. Each node is one of the 15 signal subcategories.

### Property Lifecycle

```sql l2_prop
select
  source_subcategory,
  target_subcategory,
  sum(transition_count) as transition_count,
  round(avg(median_days_between), 0) as median_days,
  round(avg(avg_days_between), 0) as avg_days
from silver_instrument_transitions_property
group by source_subcategory, target_subcategory
order by transition_count desc
```

{% sankey_chart
    data="l2_prop"
    source="source_subcategory"
    target="target_subcategory"
    value="sum(transition_count)"
    title="Property: Subcategory-to-Subcategory Flows"
/%}

{% table data="l2_prop" page_size=200 /%}

### Party Lifecycle

```sql l2_party
select
  source_subcategory,
  target_subcategory,
  sum(transition_count) as transition_count,
  round(avg(median_days_between), 0) as median_days,
  round(avg(avg_days_between), 0) as avg_days
from silver_instrument_transitions_party
group by source_subcategory, target_subcategory
order by transition_count desc
```

{% sankey_chart
    data="l2_party"
    source="source_subcategory"
    target="target_subcategory"
    value="sum(transition_count)"
    title="Party: Subcategory-to-Subcategory Flows"
/%}

{% table data="l2_party" page_size=200 /%}

### Combined (Property + Party)

```sql l2_combined
select
  source_subcategory,
  target_subcategory,
  sum(transition_count) as transition_count,
  round(avg(median_days_between), 0) as median_days,
  round(avg(avg_days_between), 0) as avg_days
from silver_instrument_transitions_combined
group by source_subcategory, target_subcategory
order by transition_count desc
```

{% sankey_chart
    data="l2_combined"
    source="source_subcategory"
    target="target_subcategory"
    value="sum(transition_count)"
    title="Combined: Subcategory-to-Subcategory Flows"
/%}

{% table data="l2_combined" page_size=200 /%}

## Level 3: Instrument-Level Transitions by Subcategory

Each section below isolates one signal subcategory and shows every instrument-to-instrument transition originating from that family. Horizontal bar charts show the top 15 transitions by volume. Tables show the top 25 with timing detail. Property linkage (what happens next on the same parcel) is the primary view. Sections ordered by total transition volume.

### Transaction Timing Triggers

Instruments signaling ownership transfers, sales, and conveyances--the core transactional lifecycle on a property.

```sql ttt_prop
select
  source_display_name || ' -> ' || target_display_name as transition,
  source_display_name,
  target_display_name,
  transition_count,
  round(median_days_between, 1) as median_days,
  round(avg_days_between, 0) as avg_days
from silver_instrument_transitions_property
where source_subcategory = 'Transaction Timing Triggers'
order by transition_count desc
limit 25
```

```sql ttt_party
select
  source_display_name || ' -> ' || target_display_name as transition,
  transition_count,
  round(median_days_between, 1) as median_days,
  round(avg_days_between, 0) as avg_days
from silver_instrument_transitions_party
where source_subcategory = 'Transaction Timing Triggers'
order by transition_count desc
limit 15
```

{% print_group %}

**Property Transitions (Top 15)**

{% horizontal_bar_chart
    data="ttt_prop"
    y="transition"
    x="sum(transition_count)"
    limit=15
    title="Transaction Timing Triggers: Property"
/%}

{% table data="ttt_prop" page_size=25 /%}

**Party Transitions (Top 15)**

{% horizontal_bar_chart
    data="ttt_party"
    y="transition"
    x="sum(transition_count)"
    title="Transaction Timing Triggers: Party"
/%}

{% /print_group %}

---

### Capital & Leverage Intelligence

Debt structure instruments: mortgages, modifications, subordinations, promissory notes. Borrowers restructuring debt are active lending targets.

```sql cli_prop
select
  source_display_name || ' -> ' || target_display_name as transition,
  source_display_name,
  target_display_name,
  transition_count,
  round(median_days_between, 1) as median_days,
  round(avg_days_between, 0) as avg_days
from silver_instrument_transitions_property
where source_subcategory = 'Capital & Leverage Intelligence'
order by transition_count desc
limit 25
```

```sql cli_party
select
  source_display_name || ' -> ' || target_display_name as transition,
  transition_count,
  round(median_days_between, 1) as median_days,
  round(avg_days_between, 0) as avg_days
from silver_instrument_transitions_party
where source_subcategory = 'Capital & Leverage Intelligence'
order by transition_count desc
limit 15
```

{% print_group %}

**Property Transitions (Top 15)**

{% horizontal_bar_chart
    data="cli_prop"
    y="transition"
    x="sum(transition_count)"
    limit=15
    title="Capital & Leverage Intelligence: Property"
/%}

{% table data="cli_prop" page_size=25 /%}

**Party Transitions (Top 15)**

{% horizontal_bar_chart
    data="cli_party"
    y="transition"
    x="sum(transition_count)"
    title="Capital & Leverage Intelligence: Party"
/%}

{% /print_group %}

---

### Title and Ownership Complexity

Instruments affecting title clarity: affidavits, amendments, powers of attorney, trust documents.

```sql toc_prop
select
  source_display_name || ' -> ' || target_display_name as transition,
  source_display_name,
  target_display_name,
  transition_count,
  round(median_days_between, 1) as median_days,
  round(avg_days_between, 0) as avg_days
from silver_instrument_transitions_property
where source_subcategory = 'Title and Ownership Complexity'
order by transition_count desc
limit 25
```

```sql toc_party
select
  source_display_name || ' -> ' || target_display_name as transition,
  transition_count,
  round(median_days_between, 1) as median_days,
  round(avg_days_between, 0) as avg_days
from silver_instrument_transitions_party
where source_subcategory = 'Title and Ownership Complexity'
order by transition_count desc
limit 15
```

{% print_group %}

**Property Transitions (Top 15)**

{% horizontal_bar_chart
    data="toc_prop"
    y="transition"
    x="sum(transition_count)"
    limit=15
    title="Title and Ownership Complexity: Property"
/%}

{% table data="toc_prop" page_size=25 /%}

**Party Transitions (Top 15)**

{% horizontal_bar_chart
    data="toc_party"
    y="transition"
    x="sum(transition_count)"
    title="Title and Ownership Complexity: Party"
/%}

{% /print_group %}

---

### Equity Unlock Signals

Release and satisfaction instruments: releases of mortgage, satisfactions, discharges. Signals a lien clearing--often followed by new lending.

```sql eus_prop
select
  source_display_name || ' -> ' || target_display_name as transition,
  source_display_name,
  target_display_name,
  transition_count,
  round(median_days_between, 1) as median_days,
  round(avg_days_between, 0) as avg_days
from silver_instrument_transitions_property
where source_subcategory = 'Equity Unlock Signals'
order by transition_count desc
limit 25
```

```sql eus_party
select
  source_display_name || ' -> ' || target_display_name as transition,
  transition_count,
  round(median_days_between, 1) as median_days,
  round(avg_days_between, 0) as avg_days
from silver_instrument_transitions_party
where source_subcategory = 'Equity Unlock Signals'
order by transition_count desc
limit 15
```

{% print_group %}

**Property Transitions (Top 15)**

{% horizontal_bar_chart
    data="eus_prop"
    y="transition"
    x="sum(transition_count)"
    limit=15
    title="Equity Unlock Signals: Property"
/%}

{% table data="eus_prop" page_size=25 /%}

**Party Transitions (Top 15)**

{% horizontal_bar_chart
    data="eus_party"
    y="transition"
    x="sum(transition_count)"
    title="Equity Unlock Signals: Party"
/%}

{% /print_group %}

---

### Life and Legal Triggers

Probate, divorce, death filings, guardianship, distribution orders. Life events that force or enable property transactions.

```sql llt_prop
select
  source_display_name || ' -> ' || target_display_name as transition,
  source_display_name,
  target_display_name,
  transition_count,
  round(median_days_between, 1) as median_days,
  round(avg_days_between, 0) as avg_days
from silver_instrument_transitions_property
where source_subcategory = 'Life and Legal Triggers'
order by transition_count desc
limit 25
```

```sql llt_party
select
  source_display_name || ' -> ' || target_display_name as transition,
  transition_count,
  round(median_days_between, 1) as median_days,
  round(avg_days_between, 0) as avg_days
from silver_instrument_transitions_party
where source_subcategory = 'Life and Legal Triggers'
order by transition_count desc
limit 15
```

{% print_group %}

**Property Transitions (Top 15)**

{% horizontal_bar_chart
    data="llt_prop"
    y="transition"
    x="sum(transition_count)"
    limit=15
    title="Life and Legal Triggers: Property"
/%}

{% table data="llt_prop" page_size=25 /%}

**Party Transitions (Top 15)**

{% horizontal_bar_chart
    data="llt_party"
    y="transition"
    x="sum(transition_count)"
    title="Life and Legal Triggers: Party"
/%}

{% /print_group %}

---

### Title Registry

Certificate of Title recordings--the foundation document linking instruments to legal parcels.

```sql tr_prop
select
  source_display_name || ' -> ' || target_display_name as transition,
  source_display_name,
  target_display_name,
  transition_count,
  round(median_days_between, 1) as median_days,
  round(avg_days_between, 0) as avg_days
from silver_instrument_transitions_property
where source_subcategory = 'Title Registry'
order by transition_count desc
limit 25
```

```sql tr_party
select
  source_display_name || ' -> ' || target_display_name as transition,
  transition_count,
  round(median_days_between, 1) as median_days,
  round(avg_days_between, 0) as avg_days
from silver_instrument_transitions_party
where source_subcategory = 'Title Registry'
order by transition_count desc
limit 15
```

{% print_group %}

**Property Transitions (Top 15)**

{% horizontal_bar_chart
    data="tr_prop"
    y="transition"
    x="sum(transition_count)"
    limit=15
    title="Title Registry: Property"
/%}

{% table data="tr_prop" page_size=25 /%}

**Party Transitions (Top 15)**

{% horizontal_bar_chart
    data="tr_party"
    y="transition"
    x="sum(transition_count)"
    title="Title Registry: Party"
/%}

{% /print_group %}

---

### Deal Complexity Events

Assignment and lease instruments that add complexity to the deal chain.

```sql dce_prop
select
  source_display_name || ' -> ' || target_display_name as transition,
  source_display_name,
  target_display_name,
  transition_count,
  round(median_days_between, 1) as median_days,
  round(avg_days_between, 0) as avg_days
from silver_instrument_transitions_property
where source_subcategory = 'Deal Complexity Events'
order by transition_count desc
limit 25
```

```sql dce_party
select
  source_display_name || ' -> ' || target_display_name as transition,
  transition_count,
  round(median_days_between, 1) as median_days,
  round(avg_days_between, 0) as avg_days
from silver_instrument_transitions_party
where source_subcategory = 'Deal Complexity Events'
order by transition_count desc
limit 15
```

{% print_group %}

**Property Transitions (Top 15)**

{% horizontal_bar_chart
    data="dce_prop"
    y="transition"
    x="sum(transition_count)"
    limit=15
    title="Deal Complexity Events: Property"
/%}

{% table data="dce_prop" page_size=25 /%}

**Party Transitions (Top 15)**

{% horizontal_bar_chart
    data="dce_party"
    y="transition"
    x="sum(transition_count)"
    title="Deal Complexity Events: Party"
/%}

{% /print_group %}

---

### Forced Sale Signals

Foreclosure, power of sale, and judicial sale instruments. High-distress indicators.

```sql fss_prop
select
  source_display_name || ' -> ' || target_display_name as transition,
  source_display_name,
  target_display_name,
  transition_count,
  round(median_days_between, 1) as median_days,
  round(avg_days_between, 0) as avg_days
from silver_instrument_transitions_property
where source_subcategory = 'Forced Sale Signals'
order by transition_count desc
limit 25
```

```sql fss_party
select
  source_display_name || ' -> ' || target_display_name as transition,
  transition_count,
  round(median_days_between, 1) as median_days,
  round(avg_days_between, 0) as avg_days
from silver_instrument_transitions_party
where source_subcategory = 'Forced Sale Signals'
order by transition_count desc
limit 15
```

{% print_group %}

**Property Transitions (Top 15)**

{% horizontal_bar_chart
    data="fss_prop"
    y="transition"
    x="sum(transition_count)"
    limit=15
    title="Forced Sale Signals: Property"
/%}

{% table data="fss_prop" page_size=25 /%}

**Party Transitions (Top 15)**

{% horizontal_bar_chart
    data="fss_party"
    y="transition"
    x="sum(transition_count)"
    title="Forced Sale Signals: Party"
/%}

{% /print_group %}

---

### Owner Pressure Signals

Tax liens, judgments, notices of default, lis pendens. Signals of financial distress on property owners.

```sql ops_prop
select
  source_display_name || ' -> ' || target_display_name as transition,
  source_display_name,
  target_display_name,
  transition_count,
  round(median_days_between, 1) as median_days,
  round(avg_days_between, 0) as avg_days
from silver_instrument_transitions_property
where source_subcategory = 'Owner Pressure Signals'
order by transition_count desc
limit 25
```

```sql ops_party
select
  source_display_name || ' -> ' || target_display_name as transition,
  transition_count,
  round(median_days_between, 1) as median_days,
  round(avg_days_between, 0) as avg_days
from silver_instrument_transitions_party
where source_subcategory = 'Owner Pressure Signals'
order by transition_count desc
limit 15
```

{% print_group %}

**Property Transitions (Top 15)**

{% horizontal_bar_chart
    data="ops_prop"
    y="transition"
    x="sum(transition_count)"
    limit=15
    title="Owner Pressure Signals: Property"
/%}

{% table data="ops_prop" page_size=25 /%}

**Party Transitions (Top 15)**

{% horizontal_bar_chart
    data="ops_party"
    y="transition"
    x="sum(transition_count)"
    title="Owner Pressure Signals: Party"
/%}

{% /print_group %}

---

### Administrative

Permits, designations, assessments, and other administrative recordings.

```sql adm_prop
select
  source_display_name || ' -> ' || target_display_name as transition,
  source_display_name,
  target_display_name,
  transition_count,
  round(median_days_between, 1) as median_days,
  round(avg_days_between, 0) as avg_days
from silver_instrument_transitions_property
where source_subcategory = 'Administrative'
order by transition_count desc
limit 25
```

```sql adm_party
select
  source_display_name || ' -> ' || target_display_name as transition,
  transition_count,
  round(median_days_between, 1) as median_days,
  round(avg_days_between, 0) as avg_days
from silver_instrument_transitions_party
where source_subcategory = 'Administrative'
order by transition_count desc
limit 15
```

{% print_group %}

**Property Transitions (Top 15)**

{% horizontal_bar_chart
    data="adm_prop"
    y="transition"
    x="sum(transition_count)"
    limit=15
    title="Administrative: Property"
/%}

{% table data="adm_prop" page_size=25 /%}

**Party Transitions (Top 15)**

{% horizontal_bar_chart
    data="adm_party"
    y="transition"
    x="sum(transition_count)"
    title="Administrative: Party"
/%}

{% /print_group %}

---

### Blocker Clearance Signals

Cancellations, terminations, and satisfaction of liens. Instruments that clear encumbrances.

```sql bcs_prop
select
  source_display_name || ' -> ' || target_display_name as transition,
  source_display_name,
  target_display_name,
  transition_count,
  round(median_days_between, 1) as median_days,
  round(avg_days_between, 0) as avg_days
from silver_instrument_transitions_property
where source_subcategory = 'Blocker Clearance Signals'
order by transition_count desc
limit 25
```

```sql bcs_party
select
  source_display_name || ' -> ' || target_display_name as transition,
  transition_count,
  round(median_days_between, 1) as median_days,
  round(avg_days_between, 0) as avg_days
from silver_instrument_transitions_party
where source_subcategory = 'Blocker Clearance Signals'
order by transition_count desc
limit 15
```

{% print_group %}

**Property Transitions (Top 15)**

{% horizontal_bar_chart
    data="bcs_prop"
    y="transition"
    x="sum(transition_count)"
    limit=15
    title="Blocker Clearance Signals: Property"
/%}

{% table data="bcs_prop" page_size=25 /%}

**Party Transitions (Top 15)**

{% horizontal_bar_chart
    data="bcs_party"
    y="transition"
    x="sum(transition_count)"
    title="Blocker Clearance Signals: Party"
/%}

{% /print_group %}

---

### Supply Signals

Maps, subdivisions, and construction-related instruments signaling new inventory entering the market.

```sql ss_prop
select
  source_display_name || ' -> ' || target_display_name as transition,
  source_display_name,
  target_display_name,
  transition_count,
  round(median_days_between, 1) as median_days,
  round(avg_days_between, 0) as avg_days
from silver_instrument_transitions_property
where source_subcategory = 'Supply Signals'
order by transition_count desc
limit 25
```

```sql ss_party
select
  source_display_name || ' -> ' || target_display_name as transition,
  transition_count,
  round(median_days_between, 1) as median_days,
  round(avg_days_between, 0) as avg_days
from silver_instrument_transitions_party
where source_subcategory = 'Supply Signals'
order by transition_count desc
limit 15
```

{% print_group %}

**Property Transitions (Top 15)**

{% horizontal_bar_chart
    data="ss_prop"
    y="transition"
    x="sum(transition_count)"
    limit=15
    title="Supply Signals: Property"
/%}

{% table data="ss_prop" page_size=25 /%}

**Party Transitions (Top 15)**

{% horizontal_bar_chart
    data="ss_party"
    y="transition"
    x="sum(transition_count)"
    title="Supply Signals: Party"
/%}

{% /print_group %}

---

### Regulatory, Risk & Claims Intelligence

Regulatory filings, environmental liens, and risk-related instruments.

```sql rrci_prop
select
  source_display_name || ' -> ' || target_display_name as transition,
  source_display_name,
  target_display_name,
  transition_count,
  round(median_days_between, 1) as median_days,
  round(avg_days_between, 0) as avg_days
from silver_instrument_transitions_property
where source_subcategory = 'Regulatory, Risk & Claims Intelligence'
order by transition_count desc
limit 25
```

```sql rrci_party
select
  source_display_name || ' -> ' || target_display_name as transition,
  transition_count,
  round(median_days_between, 1) as median_days,
  round(avg_days_between, 0) as avg_days
from silver_instrument_transitions_party
where source_subcategory = 'Regulatory, Risk & Claims Intelligence'
order by transition_count desc
limit 15
```

{% print_group %}

**Property Transitions (Top 15)**

{% horizontal_bar_chart
    data="rrci_prop"
    y="transition"
    x="sum(transition_count)"
    limit=15
    title="Regulatory, Risk & Claims Intelligence: Property"
/%}

{% table data="rrci_prop" page_size=25 /%}

**Party Transitions (Top 15)**

{% horizontal_bar_chart
    data="rrci_party"
    y="transition"
    x="sum(transition_count)"
    title="Regulatory, Risk & Claims Intelligence: Party"
/%}

{% /print_group %}

---

### Deal Propensity Events

Rescissions and amendments--instruments that signal active deal negotiation or modification.

```sql dpe_prop
select
  source_display_name || ' -> ' || target_display_name as transition,
  source_display_name,
  target_display_name,
  transition_count,
  round(median_days_between, 1) as median_days,
  round(avg_days_between, 0) as avg_days
from silver_instrument_transitions_property
where source_subcategory = 'Deal Propensity Events'
order by transition_count desc
limit 25
```

```sql dpe_party
select
  source_display_name || ' -> ' || target_display_name as transition,
  transition_count,
  round(median_days_between, 1) as median_days,
  round(avg_days_between, 0) as avg_days
from silver_instrument_transitions_party
where source_subcategory = 'Deal Propensity Events'
order by transition_count desc
limit 15
```

{% print_group %}

**Property Transitions (Top 15)**

{% horizontal_bar_chart
    data="dpe_prop"
    y="transition"
    x="sum(transition_count)"
    limit=15
    title="Deal Propensity Events: Property"
/%}

{% table data="dpe_prop" page_size=25 /%}

**Party Transitions (Top 15)**

{% horizontal_bar_chart
    data="dpe_party"
    y="transition"
    x="sum(transition_count)"
    title="Deal Propensity Events: Party"
/%}

{% /print_group %}

---

### Liens and Encumbrances

Lien instruments that create encumbrances against property. Smallest subcategory by volume.

```sql le_prop
select
  source_display_name || ' -> ' || target_display_name as transition,
  source_display_name,
  target_display_name,
  transition_count,
  round(median_days_between, 1) as median_days,
  round(avg_days_between, 0) as avg_days
from silver_instrument_transitions_property
where source_subcategory = 'Liens and Encumbrances'
order by transition_count desc
limit 25
```

```sql le_party
select
  source_display_name || ' -> ' || target_display_name as transition,
  transition_count,
  round(median_days_between, 1) as median_days,
  round(avg_days_between, 0) as avg_days
from silver_instrument_transitions_party
where source_subcategory = 'Liens and Encumbrances'
order by transition_count desc
limit 15
```

{% print_group %}

**Property Transitions (Top 15)**

{% horizontal_bar_chart
    data="le_prop"
    y="transition"
    x="sum(transition_count)"
    limit=15
    title="Liens and Encumbrances: Property"
/%}

{% table data="le_prop" page_size=25 /%}

**Party Transitions (Top 15)**

{% horizontal_bar_chart
    data="le_party"
    y="transition"
    x="sum(transition_count)"
    title="Liens and Encumbrances: Party"
/%}

{% /print_group %}

---

## Appendix: Data Sources

- **Property transitions:** silver_instrument_transitions_property -- partitioned by legal_unit_id, LEAD() ordered by date_recorded + id
- **Party transitions:** silver_instrument_transitions_party -- partitioned by party_id, LEAD() ordered by date_recorded + id
- **Combined transitions:** silver_instrument_transitions_combined -- UNION ALL of property and party with linkage_type tag
- **Classification:** dim.signal_category -- 6 categories, 15 subcategories, 165 instrument type mappings
- **Display names:** dim.dictionary -- COALESCE(sub_type display, type display, raw code)

**Note:** Subcategory prose descriptions are editorial summaries based on domain understanding, not verified against the specific instrument types classified in each subcategory.