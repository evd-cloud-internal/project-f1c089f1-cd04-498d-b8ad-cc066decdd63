---
name: 08 - Rate Gap Portfolio
assetId: 9b70d8ff-0494-4c75-8c95-001abbb66141
type: page
---

# Rate Gap Portfolio

What does today's interest rate environment mean for the mortgage book? This page sizes the opportunity by rate tier and product type, then profiles the portfolio investors who may need commercial lending products.

---

## Reference: How to Read This Page

### Rate Gap Opportunity Tiers

Every active mortgage is compared against the rate environment when it was originated and the current Guam effective rate. The gap equals origination rate minus current rate. A positive gap means the borrower is paying more than today's rate.

| Tier | Gap Threshold | What It Means |
|---|---|---|
| **STRONG_REFI** | >= +1.50% (150+ bps) | Clear refi case. Savings easily justify closing costs. Lead with rate reduction pitch. |
| **MILD_REFI** | >= +0.50% (50-149 bps) | Marginal savings. Refi pencils out only if closing costs are low or borrower also wants cash-out or term change. |
| **NEUTRAL** | -0.50% to +0.50% | No rate-driven action. Borrower is near market. Only refi for product change (ARM to fixed, etc.). |
| **RETENTION** | <= -0.50% (-50 to -149 bps) | Borrower has below-market rate. Not leaving voluntarily. Protect this relationship. |
| **STRONG_RETENTION** | <= -1.50% (150+ bps below) | Significantly cheap loan. Zero incentive to move. Highest retention priority. |
| **UNKNOWN** | NULL origination rate | No rate data for origination month. Cannot classify. |

**Caveat:** Guam effective rate is inferred from a single lender snapshot (First Hawaiian Bank, Feb 2026, -33bps spread vs national 30yr). Bank partnership data replaces this.

### Refi Sweet Spot vs Rate Gap

These answer different questions. **Rate gap** asks: *does refinancing save this borrower money given today's rates?* That's the financial answer. **Refi sweet spot** (24-60 months seasoned, active) is a conventional sales prospecting heuristic -- the window where loan officers typically target outreach. It is not an eligibility gate; borrowers can refinance at any age, and government streamline programs qualify as early as 6-7 months. The overlap -- rate-qualified AND in the prospecting window -- is where the most actionable leads live. The `Refi Sweet Spot` and `Govt Streamline` columns below show that overlap per tier.

### Portfolio Tier (Commercial Lending)

Counts distinct active mortgages per individual (non-institutional) party.

| Tier | Rule | What It Means |
|---|---|---|
| **LARGE PORTFOLIO** | 5+ active mortgages | Serious real estate investor. Commercial lending relationship -- portfolio loans, blanket mortgages, commercial lines of credit. |
| **PORTFOLIO** | 3-4 active mortgages | Active investor scaling up. May be outgrowing residential products. |
| **MULTI-PROPERTY** | exactly 2 | Second property owner -- rental, family, or investment. Entry point for commercial conversation. |

Single-mortgage borrowers are excluded from this page (filtered to 2+ active mortgages).

---

## What Does Today's Rate Environment Mean?

{% row %}
    {% big_value
        data="gold_rate_gap_portfolio_summary"
        value="sum(active_count)"
        where="detail_level = 'ALL_TYPES'"
        title="Active Mortgages with Rate Data"
        fmt="num0"
    /%}
    {% big_value
        data="gold_rate_gap_portfolio_summary"
        value="sum(active_principal)"
        where="detail_level = 'ALL_TYPES'"
        title="Active Principal"
        fmt="usd0"
    /%}
    {% big_value
        data="gold_rate_gap_portfolio_summary"
        value="sum(refi_sweet_spot_count)"
        where="detail_level = 'ALL_TYPES'"
        title="Also in Refi Sweet Spot"
        fmt="num0"
    /%}
    {% big_value
        data="gold_rate_gap_portfolio_summary"
        value="sum(govt_streamline_count)"
        where="detail_level = 'ALL_TYPES'"
        title="Govt Streamline Eligible"
        fmt="num0"
    /%}
{% /row %}

### Opportunity by Rate Tier

```sql tier_rollup
select
    rate_gap_opportunity as "Rate Tier",
    active_count as "Active Mortgages",
    active_principal as "Active Principal",
    wtd_avg_origination_rate as "Avg Origination Rate",
    wtd_avg_rate_gap as "Avg Rate Gap",
    refi_sweet_spot_count as "Refi Sweet Spot",
    govt_streamline_count as "Govt Streamline",
    pct_of_active_principal as "% of Portfolio"
from gold_rate_gap_portfolio_summary
where detail_level = 'ALL_TYPES'
order by rate_gap_opportunity
```

{% bar_chart
    data="gold_rate_gap_portfolio_summary"
    x="rate_gap_opportunity"
    y="active_principal"
    where="detail_level = 'ALL_TYPES'"
    y_fmt="usd0"
    title="Active Principal by Rate Gap Tier"
    subtitle="STRONG_REFI and MILD_REFI = borrowers paying more than today's rate"
    chart_options={
        color_palette = ["#006BA4", "#FF800E", "#5F9ED1", "#C85200", "#595959"]
    }
/%}

{% table data="tier_rollup" /%}

### Detail by Tier and Product Type

```sql tier_by_type
select
    rate_gap_opportunity as "Rate Tier",
    mortgage_type_group as "Product Type",
    mortgage_count as "Total Mortgages",
    active_count as "Active",
    active_principal as "Active Principal",
    wtd_avg_origination_rate as "Avg Orig Rate",
    wtd_avg_rate_gap as "Avg Gap",
    refi_sweet_spot_count as "Sweet Spot",
    govt_streamline_count as "Govt Streamline",
    pct_of_active_principal as "% of Portfolio"
from gold_rate_gap_portfolio_summary
where detail_level = 'BY_TYPE'
order by rate_gap_opportunity, mortgage_type_group
```

{% table data="tier_by_type" /%}

---

## Who Are Your Portfolio Investors?

{% row %}
    {% big_value
        data="gold_commercial_lending_dashboard"
        value="sum(party_count)"
        title="Portfolio Investors"
        fmt="num0"
    /%}
    {% big_value
        data="gold_commercial_lending_dashboard"
        value="sum(total_mortgage_exposure)"
        title="Total Mortgage Exposure"
        fmt="usd0"
    /%}
    {% big_value
        data="gold_commercial_lending_dashboard"
        value="sum(parties_with_contact)"
        title="With Contact Info"
        fmt="num0"
    /%}
{% /row %}

{% bar_chart
    data="gold_commercial_lending_dashboard"
    x="portfolio_tier"
    y="total_mortgage_exposure"
    y_fmt="usd0"
    title="Total Mortgage Exposure by Portfolio Tier"
    chart_options={
        color_palette = ["#006BA4", "#FF800E", "#5F9ED1", "#C85200"]
    }
/%}

```sql commercial_detail
select
    portfolio_tier as "Tier",
    party_count as "Investors",
    total_active_mortgages as "Active Mortgages",
    total_properties as "Properties",
    total_mortgage_exposure as "Total Exposure",
    avg_portfolio_value as "Avg Portfolio Value",
    avg_years_as_borrower as "Avg Years as Borrower",
    parties_with_contact as "With Contact"
from gold_commercial_lending_dashboard
order by total_mortgage_exposure desc nulls last
```

{% table data="commercial_detail" /%}