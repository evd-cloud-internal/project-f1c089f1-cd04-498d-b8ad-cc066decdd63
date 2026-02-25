---
name: 14 - Lending Work Queue
assetId: 355ef862-807b-4f9e-8f2d-b52712fc0c28
type: page
---

# Lending Work Queues

Five operational queues for the lending partnership. Every row is a callable lead or a compliance data point. The boardroom summary is on the Rate Gap Portfolio page--this page is the bullpen.

---

## Pipeline at a Glance

{% row %}
    {% big_value
        data="report_rate_gap_work_queue"
        value="count(*)"
        title="Active Mortgages"
        fmt="num0"
    /%}
    {% big_value
        data="report_rate_gap_work_queue"
        value="sum(principal)"
        title="Total Principal Under Management"
        fmt="usd0"
    /%}
    {% big_value
        data="report_rate_gap_work_queue"
        value="count(*)"
        where="rate_gap_opportunity IN ('STRONG_REFI', 'MILD_REFI')"
        title="Refi-Eligible Mortgages"
        fmt="num0"
    /%}
    {% big_value
        data="report_rate_gap_work_queue"
        value="sum(rate_gap_dollar_opportunity)"
        where="rate_gap_opportunity IN ('STRONG_REFI', 'MILD_REFI')"
        title="Total Refi Dollar Opportunity"
        fmt="usd0"
    /%}
{% /row %}

{% row %}
    {% big_value
        data="report_equity_growth_leads"
        value="count(*)"
        title="Equity Growth Properties"
        fmt="num0"
    /%}
    {% big_value
        data="report_alienation_refi_leads"
        value="count(*)"
        title="Alienation Leads"
        fmt="num0"
    /%}
    {% big_value
        data="report_commercial_lending_leads"
        value="count(*)"
        title="Portfolio Investors"
        fmt="num0"
    /%}
    {% big_value
        data="report_commercial_lending_leads"
        value="sum(total_mortgage_amount)"
        title="Portfolio Investor Exposure"
        fmt="usd0"
    /%}
{% /row %}

---

## Rate-Qualified Refi Targets

The rate-aware call sheet. Each mortgage is compared against its origination rate and today's Guam effective rate. Sorted by dollar-weighted opportunity so the biggest savings surface first.

{% row %}
    {% big_value
        data="report_rate_gap_work_queue"
        value="count(*)"
        where="rate_gap_opportunity = 'STRONG_REFI'"
        title="Strong Refi"
        fmt="num0"
    /%}
    {% big_value
        data="report_rate_gap_work_queue"
        value="sum(rate_gap_dollar_opportunity)"
        where="rate_gap_opportunity = 'STRONG_REFI'"
        title="Strong Refi $ Opportunity"
        fmt="usd0"
    /%}
    {% big_value
        data="report_rate_gap_work_queue"
        value="count(*)"
        where="rate_gap_opportunity = 'MILD_REFI'"
        title="Mild Refi"
        fmt="num0"
    /%}
    {% big_value
        data="report_rate_gap_work_queue"
        value="sum(rate_gap_dollar_opportunity)"
        where="rate_gap_opportunity = 'MILD_REFI'"
        title="Mild Refi $ Opportunity"
        fmt="usd0"
    /%}
{% /row %}

### By Rate Gap Tier

```sql rate_gap_tiers
select
  case rate_gap_opportunity
    when 'STRONG_REFI'      then '1 - STRONG REFI'
    when 'MILD_REFI'        then '2 - MILD REFI'
    when 'NEUTRAL'          then '3 - NEUTRAL'
    when 'RETENTION'        then '4 - RETENTION'
    when 'STRONG_RETENTION' then '5 - STRONG RETENTION'
  end as tier,
  count(*) as mortgages,
  sum(principal) as total_principal,
  sum(rate_gap_dollar_opportunity) as total_dollar_opportunity
from report_rate_gap_work_queue
group by rate_gap_opportunity
order by tier
```

> **STRONG REFI** = 150+ bps above current rate.  
**MILD REFI** = 50-149 bps.  
**NEUTRAL** = near market.  
**RETENTION** = 50-149 bps below.  
**STRONG RETENTION** = 150+ bps below.

{% bar_chart
    data="rate_gap_tiers"
    x="tier"
    y="sum(mortgages)"
    y_fmt="num0"
    title="Mortgages by Rate Gap Tier"
    subtitle="Left = highest urgency."
/%}

{% bar_chart
    data="rate_gap_tiers"
    x="tier"
    y="sum(total_dollar_opportunity)"
    y_fmt="usd0"
    title="Dollar Opportunity by Rate Gap Tier"
    subtitle="Negative values = retention cost (borrower has below-market rate)."
/%}

### Streamline Eligibility

The fastest closes happen where a rate gap meets a streamline product. This crosswalk shows how many refi-tier mortgages have a fast-track path.

```sql eligibility_crosswalk
select
  case rate_gap_opportunity
    when 'STRONG_REFI'      then '1 - STRONG REFI'
    when 'MILD_REFI'        then '2 - MILD REFI'
    when 'NEUTRAL'          then '3 - NEUTRAL'
    when 'RETENTION'        then '4 - RETENTION'
    when 'STRONG_RETENTION' then '5 - STRONG RETENTION'
  end as tier,
  count(*) as total_mortgages,
  count(*) filter (where is_refi_sweet_spot) as sweet_spot,
  count(*) filter (where is_va_irrrl_eligible) as va_irrrl,
  count(*) filter (where is_fha_streamline_eligible) as fha_streamline,
  count(*) filter (where is_usda_streamline_eligible) as usda_streamline
from report_rate_gap_work_queue
group by rate_gap_opportunity
order by tier
```

> **Sweet Spot** = 24-60 months seasoned (conventional prospecting window).  
**VA IRRRL** = VA mortgage, 6+ months. No appraisal, no income verification.  
**FHA Streamline** = FHA mortgage, 7+ months. Reduced documentation.  
**USDA Streamline** = USDA/RHS mortgage, 6+ months. Reduced documentation.

{% table data="eligibility_crosswalk" page_size=10 /%}

### By Mortgage Product

```sql product_mix
select
  mortgage_type_group as product,
  count(*) as total_mortgages,
  count(*) filter (where rate_gap_opportunity = 'STRONG_REFI') as strong_refi,
  count(*) filter (where rate_gap_opportunity = 'MILD_REFI') as mild_refi,
  sum(principal) filter (where rate_gap_opportunity IN ('STRONG_REFI', 'MILD_REFI')) as refi_principal
from report_rate_gap_work_queue
group by mortgage_type_group
order by total_mortgages desc
```

{% horizontal_bar_chart
    data="product_mix"
    y="product"
    x="sum(strong_refi)"
    title="Strong Refi Count by Mortgage Product"
/%}

{% table data="product_mix" page_size=10 /%}

### Rate Gap Call Sheet

```sql rate_gap_calls
select
    instrument_number as "Instrument",
    mortgage_type_group as "Product",
    principal as "Principal",
    months_since_executed as "Months",
    vintage_year as "Vintage",
    rate_gap_opportunity as "Rate Tier",
    rate_gap_guam as "Gap (Guam)",
    origination_guam_rate as "Orig Rate",
    current_guam_rate as "Current Rate",
    rate_gap_dollar_opportunity as "Dollar Opportunity",
    is_refi_sweet_spot as "Sweet Spot",
    is_va_irrrl_eligible as "VA IRRRL",
    is_fha_streamline_eligible as "FHA Streamline",
    is_usda_streamline_eligible as "USDA Streamline",
    contact_name as "Contact",
    contact_phone as "Phone",
    contact_email as "Email",
    party_count as "Parties"
from report_rate_gap_work_queue
order by rate_gap_dollar_opportunity desc nulls last
```

{% table data="rate_gap_calls" page_size=25 /%}

---

## Equity Growth Leads

Properties with rising assessed values and active mortgages. Borrowers may be eligible for refi into better terms, HELOC products, or PMI removal.

**Note:** Property ID shows UUID until `pin` is added to `silver.property_valuation_trend`.

{% row %}
    {% big_value
        data="report_equity_growth_leads"
        value="count(*)"
        title="Growing-Equity Properties"
        fmt="num0"
    /%}
    {% big_value
        data="report_equity_growth_leads"
        value="avg(value_change_pct)"
        title="Avg Value Change %"
        fmt="pct1"
    /%}
    {% big_value
        data="report_equity_growth_leads"
        value="sum(estimated_equity)"
        title="Total Estimated Equity"
        fmt="usd0"
    /%}
{% /row %}

```sql equity_leads
select
    legal_unit_id as "Property (UUID)",
    current_total_value as "Current Value",
    prior_total_value as "Prior Value",
    value_change_pct as "Change %",
    equity_trend as "Trend",
    active_mortgage_count as "Mortgages",
    total_mortgage_amount as "Total Mortgage",
    estimated_equity as "Est. Equity",
    has_va_mortgage as "VA",
    has_fha_mortgage as "FHA",
    contact_name as "Contact",
    contact_phone as "Phone",
    contact_email as "Email",
    contact_city as "City"
from report_equity_growth_leads
order by estimated_equity desc nulls last
```

{% table data="equity_leads" page_size=25 /%}

---

## Alienation Refi Leads

Properties with an active mortgage AND a recent deed transfer. The new owner likely needs to refinance--they're paying on someone else's mortgage terms.

**Note:** Property ID shows UUID until `pin` is added to `silver.alienation_refi_matches`.

{% row %}
    {% big_value
        data="report_alienation_refi_leads"
        value="count(*)"
        title="Alienation Leads"
        fmt="num0"
    /%}
    {% big_value
        data="report_alienation_refi_leads"
        value="sum(mortgage_principal)"
        title="Total Mortgage Principal"
        fmt="usd0"
    /%}
    {% big_value
        data="report_alienation_refi_leads"
        value="count(*)"
        where="lead_temperature = 'WARM'"
        title="Warm Leads"
        fmt="num0"
    /%}
{% /row %}

```sql alienation_tiers
select
  case lead_temperature
    when 'WARM'    then '1 - WARM'
    when 'COOLING' then '2 - COOLING'
    when 'AGED'    then '3 - AGED'
  end as temperature,
  count(*) as leads,
  sum(mortgage_principal) as total_principal
from report_alienation_refi_leads
group by lead_temperature
order by temperature
```

> **WARM** = transfer recorded within 30 days.  
**COOLING** = 31-90 days.  
**AGED** = 90+ days.  
New owners on WARM properties are most likely still arranging financing.

{% bar_chart
    data="alienation_tiers"
    x="temperature"
    y="sum(leads)"
    y_fmt="num0"
    title="Alienation Leads by Count"
    subtitle="Left = most actionable."
/%}

{% bar_chart
    data="alienation_tiers"
    x="temperature"
    y="sum(total_principal)"
    y_fmt="usd0"
    title="Alienation Leads by Principal"
    subtitle="Dollar exposure by temperature tier."
/%}

```sql alienation_leads
select
    legal_unit_id as "Property (UUID)",
    mortgage_type_group as "Mortgage Type",
    mortgage_principal as "Principal",
    mortgage_date_executed as "Mortgage Date",
    mortgage_months_remaining as "Months Remaining",
    mortgage_closing_party as "Closing Party",
    transaction_date_recorded as "Transfer Date",
    transaction_days_since as "Days Since Transfer",
    transaction_amount as "Transfer Amount",
    deed_category as "Deed Type",
    lead_temperature as "Temperature",
    new_owner_name as "New Owner",
    new_owner_phone as "Phone",
    new_owner_email as "Email",
    new_owner_city as "City"
from report_alienation_refi_leads
order by transaction_days_since asc, mortgage_principal desc nulls last
```

{% table data="alienation_leads" page_size=25 /%}

---

## Commercial Lending Leads

Non-institutional individuals with 2+ active mortgages across different properties. Portfolio investors who may benefit from portfolio loans, blanket mortgages, or commercial lines of credit.

{% row %}
    {% big_value
        data="report_commercial_lending_leads"
        value="count(*)"
        title="Portfolio Investors"
        fmt="num0"
    /%}
    {% big_value
        data="report_commercial_lending_leads"
        value="sum(total_mortgage_amount)"
        title="Total Mortgage Exposure"
        fmt="usd0"
    /%}
    {% big_value
        data="report_commercial_lending_leads"
        value="avg(active_mortgage_count)"
        title="Avg Active Mortgages"
        fmt="num1"
    /%}
{% /row %}

```sql commercial_tiers
select
  case portfolio_tier
    when 'LARGE PORTFOLIO' then '1 - LARGE PORTFOLIO'
    when 'PORTFOLIO'       then '2 - PORTFOLIO'
    when 'MULTI-PROPERTY'  then '3 - MULTI-PROPERTY'
  end as tier,
  count(*) as investors,
  sum(total_mortgage_amount) as total_exposure
from report_commercial_lending_leads
group by portfolio_tier
order by tier
```

> **LARGE PORTFOLIO** = 5+ active mortgages.  
**PORTFOLIO** = 3-4.  
**MULTI-PROPERTY** = exactly 2.

{% bar_chart
    data="commercial_tiers"
    x="tier"
    y="sum(investors)"
    y_fmt="num0"
    title="Investors by Portfolio Tier"
/%}

{% bar_chart
    data="commercial_tiers"
    x="tier"
    y="sum(total_exposure)"
    y_fmt="usd0"
    title="Mortgage Exposure by Portfolio Tier"
/%}

```sql commercial_leads
select
    name_full as "Borrower",
    portfolio_tier as "Tier",
    active_mortgage_count as "Active Mortgages",
    distinct_property_count as "Properties",
    total_mortgage_amount as "Total Exposure",
    avg_mortgage_amount as "Avg Mortgage",
    years_as_borrower as "Years as Borrower",
    mortgage_type_list as "Mortgage Types",
    phone_number as "Phone",
    email as "Email",
    address_city as "City"
from report_commercial_lending_leads
order by total_mortgage_amount desc nulls last
```

{% table data="commercial_leads" page_size=25 /%}

---

## CRA Quarterly Narrative

Quarterly aggregation for CRA examiner narratives. Market activity, community need indicators, positive outcomes, and government program utilization.

```sql cra_narrative
select
    report_quarter as "Quarter",
    market_transaction_signals as "Market Signals",
    mortgage_origination_volume as "Origination Volume",
    households_under_pressure as "Households Under Pressure",
    forced_sale_events as "Forced Sales",
    resolved_blockers as "Resolved Blockers",
    equity_access_events as "Equity Access",
    govt_program_mortgages as "Govt Mortgages",
    govt_program_volume as "Govt Volume"
from report_cra_quarterly_narrative
order by report_quarter desc
```

{% table data="cra_narrative" page_size=25 /%}

---

## Appendix: Definitions

### Rate Gap Opportunity Tiers

Every active mortgage is compared against the rate environment when it was originated and the current Guam effective rate. The gap equals origination rate minus current rate. A positive gap means the borrower is paying more than today's rate.

| Tier | Gap Threshold | What It Means |
|---|---|---|
| **STRONG_REFI** | >= +1.50% (150+ bps) | Clear refi case. Savings easily justify closing costs. Lead with rate reduction. |
| **MILD_REFI** | >= +0.50% (50-149 bps) | Marginal savings. Refi pencils out if closing costs are low or borrower wants cash-out/term change. |
| **NEUTRAL** | -0.50% to +0.50% | No rate-driven action. Near market. Only refi for product change (ARM to fixed, etc.). |
| **RETENTION** | <= -0.50% (-50 to -149 bps) | Below-market rate. Not leaving voluntarily. Protect this relationship. |
| **STRONG_RETENTION** | <= -1.50% (150+ bps below) | Significantly cheap loan. Zero incentive to move. Highest retention priority. |

**Caveat:** Guam effective rate is inferred from a single lender snapshot (First Hawaiian Bank, Feb 2026, -33bps spread vs national 30yr). Bank partnership data replaces this.

### Eligibility Flags

| Flag | Rule | Why It Matters |
|---|---|---|
| **Refi Sweet Spot** | Active, 24-60 months seasoned | Conventional prospecting window. Not an eligibility gate--borrowers can refi at any age. Traditional outreach timing heuristic. |
| **VA IRRRL** | Active, VA mortgage, 6+ months | No appraisal, no income verification. Streamline refi. Fastest close. |
| **FHA Streamline** | Active, FHA mortgage, 7+ months | Reduced documentation. Must demonstrate net tangible benefit. |
| **USDA Streamline** | Active, USDA/RHS mortgage, 6+ months | Reduced documentation for rural housing loans. |

### Lead Temperature (Alienation)

| Temperature | What It Means |
|---|---|
| **WARM** | Recent transfer. Window open for outreach. New owner likely still arranging financing. |
| **COOLING** | Older transfer. Window narrowing. New owner may have started shopping. |
| **AGED** | Oldest transfers. New owner may have already arranged financing, but worth checking. |

### Portfolio Tier (Commercial Lending)

| Tier | Rule | What It Means |
|---|---|---|
| **LARGE PORTFOLIO** | 5+ active mortgages | Serious investor. Portfolio loans, blanket mortgages, commercial lines. |
| **PORTFOLIO** | 3-4 active mortgages | Scaling investor. May be outgrowing residential products. |
| **MULTI-PROPERTY** | exactly 2 | Second property owner. Entry point for commercial conversation. |

### Property ID Note

Some queues on this page show the internal UUID (`legal_unit_id`) instead of the human-readable property ID (`pin`, e.g. P10000505). The Equity Growth Leads and Alienation Refi Leads views need a one-line SQL fix to carry `pin` from `base.tax_assessments` through to the final SELECT. Until then, the UUID is the only property identifier available in those views.