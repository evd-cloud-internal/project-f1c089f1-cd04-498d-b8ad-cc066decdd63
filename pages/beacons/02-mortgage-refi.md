---
name: 02 - Mortgage Refi
assetId: 5361d2e5-7e7f-4f7d-b363-ddf7da16da79
type: page
---

# Mortgage & Refi Opportunity

How big is the mortgage book, what products dominate, and where are the refi opportunities? This page sizes the lending universe and identifies which product segments have the most actionable pipeline.

---

## Reference: How to Read This Page

### Mortgage Type Groups

Every mortgage is classified into a product group based on OCR extraction from the recorded instrument. These groups drive all downstream refi, rate gap, and eligibility analysis.

| Group | Products | Assumed Term | Banking Relevance |
|---|---|---|---|
| **Consumer Conventional** | Conventional mortgages, rate riders | 30yr / 5yr | Core residential book. Largest volume. Refi sweet spot and rate gap analysis targets. |
| **Government-Backed** | VA, FHA, USDA/RHS, Guam Housing Corp | 30yr | Streamline refi programs (VA IRRRL at 6mo, FHA at 7mo, USDA at 6mo). Fastest closes. |
| **Credit Line** | HELOCs, credit agreements | 10yr | HELOC reset window (8-10 years). Approaching reset = conversion opportunity. |
| **Construction** | Construction mortgages | 18mo | Construction-to-permanent conversion window (6-18 months). |
| **Commercial** | Commercial, SBA | 10yr | Portfolio lending. Cross-reference with commercial lending leads on the Lending POC page. |
| **Private** | Private mortgages | 2yr | Short-term private financing. Bank conversion candidates after 12+ months. |
| **Foreclosure-Right Structure** | Power of sale mortgages | 10yr | Structural foreclosure rights. Low volume, high distress relevance. |

### Refi Opportunity Columns

| Column | What It Means |
|---|---|
| **Refi Window** | Active mortgages 24-60 months seasoned. Conventional prospecting heuristic, not an eligibility gate. |
| **VA IRRRL Ready** | Active VA mortgages with 6+ months seasoning. Streamline refi, no appraisal needed. |
| **FHA Streamline Ready** | Active FHA mortgages with 7+ months. Reduced documentation refi. |
| **USDA Streamline Ready** | Active USDA/RHS with 6+ months. Rural housing streamline. |
| **HELOC Reset Window** | Credit lines at 96-120 months (8-10 years). Approaching draw period end. |
| **Private Conversion** | Active private mortgages at 12+ months. May be ready for bank product. |
| **Const-to-Perm** | Construction mortgages at 6-18 months. Conversion to permanent financing. |
| **Equity Rich** | Active mortgages with LTV under 60%. Prime refi candidates with substantial equity. |

---

## The Mortgage Book

{% row %}
    {% big_value
        data="gold_refi_opportunity_dashboard"
        value="sum(total_mortgages)"
        title="All Mortgages (Lifetime)"
        fmt="num0"
    /%}
    {% big_value
        data="gold_refi_opportunity_dashboard"
        value="sum(total_principal)"
        title="All Principal (Lifetime)"
        fmt="usd0"
    /%}
    {% big_value
        data="gold_rate_gap_portfolio_summary"
        value="sum(active_count)"
        where="detail_level = 'ALL_TYPES'"
        title="Likely Active"
        fmt="num0"
    /%}
    {% big_value
        data="gold_rate_gap_portfolio_summary"
        value="sum(active_principal)"
        where="detail_level = 'ALL_TYPES'"
        title="Active Principal"
        fmt="usd0"
    /%}
{% /row %}
{% row %}
    {% big_value
        data="gold_refi_opportunity_dashboard"
        value="sum(va_irrrl_ready)"
        title="VA IRRRL Ready"
        fmt="num0"
    /%}
    {% big_value
        data="gold_refi_opportunity_dashboard"
        value="sum(fha_streamline_ready)"
        title="FHA Streamline Ready"
        fmt="num0"
    /%}
    {% big_value
        data="gold_refi_opportunity_dashboard"
        value="sum(usda_streamline_ready)"
        title="USDA Streamline Ready"
        fmt="num0"
    /%}
    {% big_value
        data="gold_refi_opportunity_dashboard"
        value="sum(heloc_reset_window)"
        title="HELOC Reset Approaching"
        fmt="num0"
    /%}
{% /row %}
{% row %}
    {% big_value
        data="gold_refi_opportunity_dashboard"
        value="sum(refi_window_count)"
        title="In Refi Window (24-60mo)"
        fmt="num0"
    /%}
        {% big_value
        data="gold_rate_gap_portfolio_summary"
        value="sum(active_count)"
        where="detail_level = 'ALL_TYPES' AND (rate_gap_opportunity = 'STRONG_REFI' OR rate_gap_opportunity = 'MILD_REFI')"
        title="All Rate-Qualified Refi"
        fmt="num0"
    /%}
    {% big_value
        data="gold_rate_gap_portfolio_summary"
        value="sum(active_count)"
        where="detail_level = 'ALL_TYPES' AND rate_gap_opportunity = 'STRONG_REFI'"
        title="Strong Refi (150+ bps)"
        fmt="num0"
    /%}
    {% big_value
        data="gold_rate_gap_portfolio_summary"
        value="sum(active_count)"
        where="detail_level = 'ALL_TYPES' AND rate_gap_opportunity = 'MILD_REFI'"
        title="Mild Refi (50-149 bps)"
        fmt="num0"
    /%}
{% /row %}

### Rate-Qualified Opportunity by Product Group
```sql refi_by_group
select
    mortgage_type_group as "Product Group",
    sum(active_count) as "Active",
    sum(active_principal) as "Active Principal",
    sum(case when rate_gap_opportunity in ('STRONG_REFI', 'MILD_REFI')
        then active_count else 0 end) as "Rate-Qualified Refi",
    sum(case when rate_gap_opportunity in ('STRONG_REFI', 'MILD_REFI')
        then active_principal else 0 end) as "Rate-Qualified $",
    sum(case when rate_gap_opportunity = 'STRONG_REFI'
        then active_count else 0 end) as "Strong Refi",
    sum(case when rate_gap_opportunity = 'MILD_REFI'
        then active_count else 0 end) as "Mild Refi",
    sum(refi_sweet_spot_count) as "Also in Sweet Spot",
    sum(govt_streamline_count) as "Govt Streamline"
from gold_rate_gap_portfolio_summary
where detail_level = 'BY_TYPE'
group by mortgage_type_group
order by sum(active_principal) desc nulls last
```

{% table data="refi_by_group" /%}

### Rate-Qualified Detail by Tier and Product
```sql refi_by_tier
select
    rate_gap_opportunity as "Rate Tier",
    mortgage_type_group as "Product Group",
    active_count as "Active",
    active_principal as "Active Principal",
    wtd_avg_origination_rate as "Avg Orig Rate",
    wtd_avg_rate_gap as "Avg Gap",
    refi_sweet_spot_count as "Sweet Spot",
    govt_streamline_count as "Govt Streamline",
    pct_of_active_principal as "% of Portfolio"
from gold_rate_gap_portfolio_summary
where detail_level = 'BY_TYPE'
order by rate_gap_opportunity, active_principal desc nulls last
```

{% table data="refi_by_tier" /%}

---

## Market Share -- Product Mix Over Time

How has the mortgage product mix shifted since 2018? Changing share tells you where the market is moving and where your product strategy needs to follow.

### By Count

{% table
    data="gold_mortgage_market_share_trends"
%}
    {% dimension
        value="mortgage_type_group"
    /%}
    {% pivot
        value="vintage_year"
    /%}
    {% measure
        value="sum(mortgage_count)"
        fmt="num0"
    /%}
{% /table %}

### By Dollar Volume

{% table
    data="gold_mortgage_market_share_trends"
    where="vintage_year >= 2020"
%}
    {% dimension
        value="mortgage_type_group"
    /%}
    {% pivot
        value="vintage_year"
    /%}
    {% measure
        value="sum(total_principal)"
        fmt="usd0"
    /%}
{% /table %}

### Market Share % by Count

{% table
    data="gold_mortgage_market_share_trends"
%}
    {% dimension
        value="mortgage_type_group"
    /%}
    {% pivot
        value="vintage_year"
    /%}
    {% measure
        value="sum(pct_of_count)"
        fmt="num1"
        title="Share %"
    /%}
{% /table %}