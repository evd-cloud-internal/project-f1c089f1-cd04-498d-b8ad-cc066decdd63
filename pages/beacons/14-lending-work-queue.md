---
name: 14 - Lending Work Queue
assetId: 355ef862-807b-4f9e-8f2d-b52712fc0c28
type: page
---

# Lending Work Queues

Five operational queues for the lending partnership. Every row is a callable lead or a compliance data point. The boardroom summary is on the Rate Gap Portfolio page -- this page is the bullpen.

---

## Reference: How to Read This Page

### Rate Gap Opportunity Tiers

Every active mortgage is compared against the rate environment when it was originated and the current Guam effective rate. The gap equals origination rate minus current rate. A positive gap means the borrower is paying more than today's rate.

| Tier | Gap Threshold | What It Means |
|---|---|---|
| **STRONG_REFI** | >= +1.50% (150+ bps) | Clear refi case. Savings easily justify closing costs. Lead with rate reduction. |
| **MILD_REFI** | >= +0.50% (50-149 bps) | Marginal savings. Refi pencils out if closing costs are low or borrower wants cash-out/term change. |
| **NEUTRAL** | -0.50% to +0.50% | No rate-driven action. Near market. Only refi for product change (ARM to fixed, etc.). |
| **RETENTION** | <= -0.50% (-50 to -149 bps) | Below-market rate. Not leaving voluntarily. Protect this relationship. |
| **STRONG_RETENTION** | <= -1.50% (150+ bps below) | Significantly cheap loan. Zero incentive to move. Highest retention priority. |
| **UNKNOWN** | NULL origination rate | No rate data for origination month. Cannot classify. |

**Caveat:** Guam effective rate is inferred from a single lender snapshot (First Hawaiian Bank, Feb 2026, -33bps spread vs national 30yr). Bank partnership data replaces this.

### Eligibility Flags on the Rate Gap Queue

The rate gap queue carries pre-computed flags alongside the rate tier. These are independent dimensions -- a mortgage can be STRONG_REFI and also VA IRRRL eligible. The overlap is where the fastest closes live.

| Flag | Rule | Why It Matters |
|---|---|---|
| **Refi Sweet Spot** | Active, 24-60 months seasoned | Conventional prospecting window. Not an eligibility gate -- borrowers can refi at any age. Traditional outreach timing heuristic. |
| **VA IRRRL** | Active, VA mortgage, 6+ months | No appraisal, no income verification. Streamline refi. Fastest close. |
| **FHA Streamline** | Active, FHA mortgage, 7+ months | Reduced documentation. Must demonstrate net tangible benefit. |
| **USDA Streamline** | Active, USDA/RHS mortgage, 6+ months | Reduced documentation for rural housing loans. |

### Lead Temperature (Alienation)

How recently the deed transfer was recorded relative to the active mortgage.

| Temperature | What It Means |
|---|---|
| **HOT** | Very recent transfer. New owner urgently needs financing. |
| **WARM** | Recent transfer. Window still open for outreach. |
| **COOL** | Older transfer. New owner may have already arranged financing. |

### Portfolio Tier (Commercial Lending)

Counts distinct active mortgages per individual (non-institutional) party.

| Tier | Rule | What It Means |
|---|---|---|
| **LARGE PORTFOLIO** | 5+ active mortgages | Serious investor. Portfolio loans, blanket mortgages, commercial lines. |
| **PORTFOLIO** | 3-4 active mortgages | Scaling investor. May be outgrowing residential products. |
| **MULTI-PROPERTY** | exactly 2 | Second property owner. Entry point for commercial conversation. |

### Property ID Note

Some queues on this page show the internal UUID (`legal_unit_id`) instead of the human-readable property ID (`pin`, e.g. P10000505). The Equity Growth Leads and Alienation Refi Leads views need a one-line SQL fix to carry `pin` from `base.tax_assessments` through to the final SELECT. Until then, the UUID is the only property identifier available in those views.

---

## Rate-Qualified Refi Targets

The rate-aware call sheet. Each mortgage is compared against its origination rate and today's Guam effective rate. Sorted by dollar-weighted opportunity so the biggest savings surface first. Eligibility flags tell the loan officer which streamline product to pitch.

{% row %}
    {% big_value
        data="report_rate_gap_work_queue"
        value="count(*)"
        title="Active Mortgages in Queue"
        fmt="num0"
    /%}
    {% big_value
        data="report_rate_gap_work_queue"
        value="sum(principal)"
        title="Total Principal"
        fmt="usd0"
    /%}
    {% big_value
        data="report_rate_gap_work_queue"
        value="count(*)"
        where="rate_gap_opportunity = 'STRONG_REFI'"
        title="Strong Refi Opportunities"
        fmt="num0"
    /%}
    {% big_value
        data="report_rate_gap_work_queue"
        value="sum(rate_gap_dollar_opportunity)"
        where="rate_gap_opportunity = 'STRONG_REFI'"
        title="Strong Refi Dollar Opportunity"
        fmt="usd0"
    /%}
{% /row %}

{% bar_chart
    data="report_rate_gap_work_queue"
    x="rate_gap_opportunity"
    y="count(*)"
    y_fmt="num0"
    title="Queue Distribution by Rate Gap Tier"
    subtitle="STRONG_REFI = 150+ bps above current rate"
    chart_options={
        color_palette = ["#006BA4", "#FF800E", "#5F9ED1", "#C85200", "#595959"]
    }
/%}

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

{% table data="rate_gap_calls" /%}

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
order by value_change_pct desc
```

{% table data="equity_leads" /%}

---

## Alienation Refi Leads

Properties with an active mortgage AND a recent deed transfer. The new owner likely needs to refinance -- they're paying on someone else's mortgage terms.

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
        where="lead_temperature = 'HOT'"
        title="Hot Leads"
        fmt="num0"
    /%}
{% /row %}

{% bar_chart
    data="report_alienation_refi_leads"
    x="lead_temperature"
    y="count(*)"
    y_fmt="num0"
    title="Alienation Leads by Temperature"
    chart_options={
        color_palette = ["#006BA4", "#FF800E", "#5F9ED1", "#C85200"]
    }
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

{% table data="alienation_leads" /%}

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

{% table data="commercial_leads" /%}

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

{% table data="cra_narrative" /%}