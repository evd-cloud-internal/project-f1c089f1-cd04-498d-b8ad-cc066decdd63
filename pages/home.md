---
name: Home
assetId: c18a30f5-1d9f-4e71-91d9-e5161bcc6957
type: page
---

# Executive Signals Overview

The pulse of Guam's real property market. Every recorded instrument -- mortgage, deed, lien, release, foreclosure notice -- is a signal. This page shows what happened in the last 30 days, how it compares to the prior 30, and the long-term trend by category.

---

## Reference: Signal Categories

RealFACTS classifies every recorded instrument from the Department of Land Management into one of six signal categories. Each category tells a different story about what's happening in the market.

### Categories and What They Mean for a Lending Partner

| Category | What It Captures | Banking Relevance |
|---|---|---|
| **Capital & Leverage Intelligence** | Mortgages, modifications, subordinations, bonds, promissory notes, assumptions | Core lending activity. New originations, restructuring, and capital stack changes. This is your book of business. |
| **Transaction Propensity Intelligence** | Deeds, leases, contracts, sales, releases, liens, judgments, probate, forced sales | The broadest category. Contains six subcategories (below) covering everything from routine sales to foreclosures. |
| **Transaction Feasibility & Close Intelligence** | Quitclaim deeds, affidavits, title certificates, assignments, lis pendens, corrections | Deal complexity indicators. Title issues, encumbrances, and documentation that affects whether a transaction can close. |
| **Market Dynamics Intelligence** | Zone changes, permits, easements, declarations of taking, bills of sale | Supply pipeline and development signals. Early indicators of new inventory entering the market. |
| **Regulatory, Risk & Claims Intelligence** | Waivers, notices of taking, designations, declarations, covenants, assessments | Government and regulatory activity. Low volume but high importance for compliance monitoring. |
| **Metadata** | Administrative filings, bylaws, articles, minutes, certificates of title | Operational noise. Minimal signal value. Included for completeness; typically filtered out of partner-facing views. |

### Transaction Propensity Intelligence -- Subcategories

This is the largest category and breaks into seven distinct signal families:

| Subcategory | What It Captures | Action |
|---|---|---|
| **Transaction Timing Triggers** | Deeds, conveyances, leases, contracts, options, maps, bills of sale | Active market transactions. Deposits pipeline, sales outreach, new relationship opportunities. |
| **Equity Unlock Signals** | Mortgage releases, satisfactions, discharges, cancellation of judgments | Borrower just freed collateral. Either refinanced with a competitor (win-back) or paid off (HELOC opportunity). |
| **Forced Sale Signals** | Notices of default/foreclosure, tax deeds, trustee deeds, power of sale deeds | Highest distress. CRA counseling targets and potential distressed acquisition opportunities. |
| **Owner Pressure Signals** | Liens, judgments, lis pendens, tax lien notices, mechanics liens, writs of execution | Financial stress short of foreclosure. Early intervention and counseling candidates. |
| **Blocker Clearance Signals** | Cancellations, terminations, withdrawals, dismissals | Positive resolution. A blocker was removed -- property may now be ready for a new transaction. |
| **Life and Legal Triggers** | Death certificates, divorce decrees, probate distributions, powers of attorney, trusts, gift deeds | Life events that force property decisions. Probate pipeline, estate planning, ownership transitions. |
| **Deal Propensity Events** | Rescissions, amendments | Deal modifications. Low volume indicator of transaction renegotiation. |

### Signal Weights

Each instrument type carries three weights (1-10 scale) that drive the scoring in downstream views:

- **Signal Weight** -- overall importance as a market indicator
- **Distress Weight** -- relevance to financial stress and CRA counseling
- **Revenue Weight** -- relevance to lending and revenue generation

These weights feed the `weighted_distress_score` and `weighted_revenue_score` on the Property Intelligence Scoreboard (Page 6).

---

## Last 30 Days at a Glance

{% row %}
    {% big_value
        data="gold_executive_signals_summary"
        value="signals_last_30_days"
        title="Total Signals (30 Days)"
        fmt="num0"
    /%}
    {% big_value
        data="gold_executive_signals_summary"
        value="transaction_propensity_last_30d"
        title="Transaction Propensity"
        fmt="num0"
    /%}
    {% big_value
        data="gold_executive_signals_summary"
        value="capital_leverage_volume_last_30d"
        title="Capital & Leverage Volume"
        fmt="usd0"
    /%}
{% /row %}

{% row %}
    {% big_value
        data="gold_executive_signals_summary"
        value="equity_unlock_last_30d"
        title="Equity Unlocks"
        fmt="num0"
    /%}
    {% big_value
        data="gold_executive_signals_summary"
        value="owner_pressure_last_30d"
        title="Owner Pressure"
        fmt="num0"
    /%}
    {% big_value
        data="gold_executive_signals_summary"
        value="forced_sale_last_30d"
        title="Forced Sales"
        fmt="num0"
    /%}
    {% big_value
        data="gold_executive_signals_summary"
        value="blocker_clearance_last_30d"
        title="Blockers Cleared"
        fmt="num0"
    /%}
{% /row %}

### 30-Day vs Prior 30-Day Comparison

```sql period_comparison
select
    'Total Signals' as metric,
    signals_last_30_days::double as current_30_days,
    signals_prior_30_days::double as prior_30_days,
    (signals_last_30_days - signals_prior_30_days)::double as delta
from gold_executive_signals_summary

union all

select
    'Owner Pressure',
    owner_pressure_last_30d::double,
    owner_pressure_prior_30d::double,
    owner_pressure_delta_30d::double
from gold_executive_signals_summary

union all

select
    'Equity Unlocks',
    equity_unlock_last_30d::double,
    equity_unlock_prior_30d::double,
    equity_unlock_delta_30d::double
from gold_executive_signals_summary

union all

select
    'Capital & Leverage Volume ($)',
    capital_leverage_volume_last_30d,
    capital_leverage_volume_prior_30d,
    capital_leverage_volume_last_30d - capital_leverage_volume_prior_30d
from gold_executive_signals_summary
```

{% table data="period_comparison" /%}

---

## Signal Velocity -- Month over Month

How is each signal category accelerating or decelerating? Rising owner pressure with falling blocker clearance is a warning. Rising equity unlocks with stable capital signals means borrowers are leaving.

### All Categories -- Last 12 Months

{% table
    data="gold_signal_velocity_trends"
    date_range={
        range="last 12 months"
        date="record_month"
    }
%}
    {% dimension
        value="category"
    /%}
    {% pivot
        value="record_month"
        date_grain="month"
    /%}
    {% measure
        value="sum(current_month_count)"
        fmt="num0"
    /%}
{% /table %}

### Capital & Leverage -- Mortgage Activity Trend

New originations, modifications, and capital stack changes. This is the core lending heartbeat.

{% line_chart
    data="gold_signal_velocity_trends"
    x="record_month"
    y="current_month_count"
    where="category = 'Capital & Leverage Intelligence'"
    y_fmt="num0"
    title="Capital & Leverage -- Monthly Volume"
    chart_options={
        color_palette = ["#006BA4"]
    }
/%}

{% line_chart
    data="gold_signal_velocity_trends"
    x="record_month"
    y="current_month_amount"
    where="category = 'Capital & Leverage Intelligence'"
    y_fmt="usd0"
    title="Capital & Leverage -- Monthly Dollar Volume"
    chart_options={
        color_palette = ["#FF800E"]
    }
/%}

### Transaction Propensity -- Where the Lending Signals Live

Seven subcategories ranging from routine sales to foreclosures. Watch equity unlocks vs owner pressure -- divergence means trouble.

{% table
    data="gold_signal_velocity_trends"
    where="category = 'Transaction Propensity Intelligence'"
    date_range={
        range="last 12 months"
        date="record_month"
    }
%}
    {% dimension
        value="subcategory"
    /%}
    {% pivot
        value="record_month"
        date_grain="month"
    /%}
    {% measure
        value="sum(current_month_count)"
        fmt="num0"
    /%}
{% /table %}

### Feasibility & Close -- Deal Complexity Indicators

Title issues, encumbrances, and lis pendens. Spikes here mean deals are getting harder to close.

{% line_chart
    data="gold_signal_velocity_trends"
    x="record_month"
    y="sum(current_month_count)"
    series="subcategory"
    where="category = 'Transaction Feasibility & Close Intelligence'"
    y_fmt="num0"
    title="Feasibility & Close Subcategories -- Monthly Volume"
    chart_options={
        color_palette = ["#006BA4", "#FF800E", "#5F9ED1"]
    }
/%}

---

## Annual Trend

Year-over-year volume by category since 2018. Use for strategic planning, compliance reporting, and multi-year trend analysis.

### All Categories -- Annual Volume

{% table
    data="gold_yearly_instrument_summary"
%}
    {% dimension
        value="category"
    /%}
    {% pivot
        value="recorded_year"
    /%}
    {% measure
        value="sum(instrument_count)"
        fmt="num0"
    /%}
{% /table %}

### Capital & Leverage -- Annual Dollar Volume

{% bar_chart
    data="gold_yearly_instrument_summary"
    x="recorded_year"
    y="sum(total_amount)"
    where="category = 'Capital & Leverage Intelligence'"
    y_fmt="usd0"
    title="Capital & Leverage -- Annual Dollar Volume"
    chart_options={
        color_palette = ["#006BA4"]
    }
/%}


### Transaction Propensity -- Annual Volume by Subcategory

{% table
    data="gold_yearly_instrument_summary"
    where="category = 'Transaction Propensity Intelligence'"
%}
    {% dimension
        value="subcategory"
    /%}
    {% pivot
        value="recorded_year"
    /%}
    {% measure
        value="sum(instrument_count)"
        fmt="num0"
    /%}
{% /table %}

---

## Pareto Analysis -- Where Do Signals Concentrate?

Not all signal types carry equal weight. A handful of subcategories drive the majority of both volume and dollar exposure. The Pareto charts below identify which ones account for 80% of the total -- those are the ones worth watching closely.

### Volume Pareto -- Last 12 Months

Which subcategories generate the most recorded instruments?

```sql pareto_volume
with subcategory_totals as (
    select
        subcategory,
        sum(current_month_count) as total_count
    from gold_signal_velocity_trends
    where record_month >= (
        select date_trunc('month', max(record_month) - interval '12' month)
        from gold_signal_velocity_trends
    )
    group by 1
),
ranked as (
    select
        subcategory,
        total_count,
        sum(total_count) over (order by total_count desc) as running_total,
        sum(total_count) over () as grand_total
    from subcategory_totals
)
select
    subcategory,
    total_count,
    round(100.0 * total_count / grand_total, 1) as pct_of_total,
    round(100.0 * running_total / grand_total, 1) as cumulative_pct,
    case when running_total - total_count < 0.8 * grand_total then 'Top 80%' else '' end as in_80
from ranked
order by total_count desc
```

{% combo_chart
    data="pareto_volume"
    x="subcategory"
    x_sort="data"
    y_fmt="num0"
    y2_fmt="num1"
    title="Signal Volume Pareto -- Last 12 Months"
    subtitle="Bars = instrument count per subcategory. Line = cumulative % of total."
%}
    {% bar
        y="total_count"
    /%}
    {% line
        y="cumulative_pct"
        axis="y2"
    /%}
{% /combo_chart %}

{% table data="pareto_volume" /%}

### Dollar Pareto -- Last 12 Months

Which subcategories carry the most dollar exposure? Volume and dollars tell different stories -- a low-volume subcategory with large dollar amounts is still a high-value signal.

```sql pareto_dollars
with subcategory_totals as (
    select
        subcategory,
        sum(current_month_amount) as total_amount
    from gold_signal_velocity_trends
    where record_month >= (
        select date_trunc('month', max(record_month) - interval '12' month)
        from gold_signal_velocity_trends
    )
    group by 1
),
ranked as (
    select
        subcategory,
        total_amount,
        sum(total_amount) over (order by total_amount desc) as running_total,
        sum(total_amount) over () as grand_total
    from subcategory_totals
)
select
    subcategory,
    total_amount,
    round(100.0 * total_amount / nullif(grand_total, 0), 1) as pct_of_total,
    round(100.0 * running_total / nullif(grand_total, 0), 1) as cumulative_pct,
    case when running_total - total_amount < 0.8 * grand_total then 'Top 80%' else '' end as in_80
from ranked
order by total_amount desc
```

{% combo_chart
    data="pareto_dollars"
    x="subcategory"
    x_sort="data"
    y_fmt="usd0"
    y2_fmt="num1"
    title="Signal Dollar Pareto -- Last 12 Months"
    subtitle="Bars = total dollar volume per subcategory. Line = cumulative % of total."
%}
    {% bar
        y="total_amount"
    /%}
    {% line
        y="cumulative_pct"
        axis="y2"
    /%}
{% /combo_chart %}

{% table data="pareto_dollars" /%}

---

## Top Subcategory Trends

The Pareto tells you which subcategories matter most. These charts track the top four by volume and by dollars over time so you can see direction, not just magnitude.

### Monthly Volume -- Top Subcategories

```sql top_volume_trend
with ranked as (
    select
        subcategory,
        sum(current_month_count) as total_count,
        row_number() over (order by sum(current_month_count) desc) as rn
    from gold_signal_velocity_trends
    where record_month >= (
        select date_trunc('month', max(record_month) - interval '12' month)
        from gold_signal_velocity_trends
    )
    group by 1
)
select
    v.record_month,
    v.subcategory,
    v.current_month_count
from gold_signal_velocity_trends v
inner join ranked r on v.subcategory = r.subcategory
where r.rn <= 4
order by v.record_month, v.subcategory
```

{% line_chart
    data="top_volume_trend"
    x="record_month"
    y="current_month_count"
    series="subcategory"
    y_fmt="num0"
    title="Top 4 Subcategories -- Monthly Instrument Count"
    subtitle="Ranked by 12-month total volume"
    chart_options={
        color_palette = ["#006BA4", "#FF800E", "#5F9ED1", "#C85200"]
    }
/%}

### Monthly Dollar Volume -- Top Subcategories

```sql top_dollar_trend
with ranked as (
    select
        subcategory,
        sum(current_month_amount) as total_amount,
        row_number() over (order by sum(current_month_amount) desc) as rn
    from gold_signal_velocity_trends
    where record_month >= (
        select date_trunc('month', max(record_month) - interval '12' month)
        from gold_signal_velocity_trends
    )
    group by 1
)
select
    v.record_month,
    v.subcategory,
    v.current_month_amount
from gold_signal_velocity_trends v
inner join ranked r on v.subcategory = r.subcategory
where r.rn <= 4
order by v.record_month, v.subcategory
```

{% line_chart
    data="top_dollar_trend"
    x="record_month"
    y="current_month_amount"
    series="subcategory"
    y_fmt="usd0"
    title="Top 4 Subcategories -- Monthly Dollar Volume"
    subtitle="Ranked by 12-month total dollar volume"
    chart_options={
        color_palette = ["#006BA4", "#FF800E", "#5F9ED1", "#C85200"]
    }
/%}

---

## Statistical Process Control -- Is the Change Real?

A month-over-month spike looks alarming but might be normal variation. These control charts apply XmR (Individuals and Moving Range) analysis to distinguish signal from noise. The center line is the process mean. The upper and lower control limits (UCL/LCL) mark +/-3 sigma -- anything inside is expected variation, anything outside is a statistically significant shift worth investigating.

Four subcategories are charted below -- the ones with the most direct relevance to lending operations. Both volume and dollar versions are shown because a volume spike in low-dollar instruments has different implications than a dollar spike in high-value mortgages.

### Capital & Leverage Intelligence

Core lending activity. A point outside the control limits means mortgage origination or restructuring activity has genuinely shifted -- not just a noisy month.

```sql control_capital_volume
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where subcategory = 'Capital & Leverage Intelligence'
    group by 1
),
with_mr as (
    select record_month, cnt,
        abs(cnt - lag(cnt) over (order by record_month)) as mr
    from monthly
),
stats as (
    select avg(cnt) as mean_val, avg(mr) / 1.128 as sigma
    from with_mr
)
select
    m.record_month,
    m.cnt as monthly_count
from monthly m
order by m.record_month
```

```sql control_capital_volume_limits
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where subcategory = 'Capital & Leverage Intelligence'
    group by 1
),
with_mr as (
    select cnt, abs(cnt - lag(cnt) over (order by record_month)) as mr
    from monthly
),
stats as (
    select avg(cnt) as mean_val, avg(mr) / 1.128 as sigma
    from with_mr
)
select round(mean_val + 3 * sigma, 0) as ref_val, 'UCL' as ref_label from stats
union all
select round(mean_val, 0), 'Mean' from stats
union all
select greatest(0, round(mean_val - 3 * sigma, 0)), 'LCL' from stats
```

{% line_chart
    data="control_capital_volume"
    x="record_month"
    y="monthly_count"
    y_fmt="num0"
    title="Capital & Leverage -- Volume Control Chart (XmR)"
    subtitle="Points outside UCL/LCL represent statistically significant shifts in lending activity"
    chart_options={
        color_palette = ["#006BA4"]
    }
%}
    {% reference_line
        data="control_capital_volume_limits"
        y="ref_val"
        label="ref_label"
        color="#595959"
        line_options={
            type = "dashed"
            width = 1
        }
    /%}
{% /line_chart %}

```sql control_capital_dollars
with monthly as (
    select record_month, sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where subcategory = 'Capital & Leverage Intelligence'
    group by 1
),
with_mr as (
    select record_month, amt,
        abs(amt - lag(amt) over (order by record_month)) as mr
    from monthly
),
stats as (
    select avg(amt) as mean_val, avg(mr) / 1.128 as sigma
    from with_mr
)
select
    m.record_month,
    m.amt as monthly_dollars
from monthly m
order by m.record_month
```

```sql control_capital_dollars_limits
with monthly as (
    select record_month, sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where subcategory = 'Capital & Leverage Intelligence'
    group by 1
),
with_mr as (
    select amt, abs(amt - lag(amt) over (order by record_month)) as mr
    from monthly
),
stats as (
    select avg(amt) as mean_val, avg(mr) / 1.128 as sigma
    from with_mr
)
select round(mean_val + 3 * sigma, 0) as ref_val, 'UCL' as ref_label from stats
union all
select round(mean_val, 0), 'Mean' from stats
union all
select greatest(0, round(mean_val - 3 * sigma, 0)), 'LCL' from stats
```

{% line_chart
    data="control_capital_dollars"
    x="record_month"
    y="monthly_dollars"
    y_fmt="usd0"
    title="Capital & Leverage -- Dollar Volume Control Chart (XmR)"
    chart_options={
        color_palette = ["#FF800E"]
    }
%}
    {% reference_line
        data="control_capital_dollars_limits"
        y="ref_val"
        label="ref_label"
        color="#595959"
        line_options={
            type = "dashed"
            width = 1
        }
    /%}
{% /line_chart %}

### Transaction Timing Triggers

Market transaction activity -- deeds, sales, conveyances. A shift here means the pace of property transactions has genuinely changed.

```sql control_transaction_volume
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where subcategory = 'Transaction Timing Triggers'
    group by 1
),
with_mr as (
    select record_month, cnt,
        abs(cnt - lag(cnt) over (order by record_month)) as mr
    from monthly
),
stats as (
    select avg(cnt) as mean_val, avg(mr) / 1.128 as sigma
    from with_mr
)
select
    m.record_month,
    m.cnt as monthly_count
from monthly m
order by m.record_month
```

```sql control_transaction_volume_limits
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where subcategory = 'Transaction Timing Triggers'
    group by 1
),
with_mr as (
    select cnt, abs(cnt - lag(cnt) over (order by record_month)) as mr
    from monthly
),
stats as (
    select avg(cnt) as mean_val, avg(mr) / 1.128 as sigma
    from with_mr
)
select round(mean_val + 3 * sigma, 0) as ref_val, 'UCL' as ref_label from stats
union all
select round(mean_val, 0), 'Mean' from stats
union all
select greatest(0, round(mean_val - 3 * sigma, 0)), 'LCL' from stats
```

{% line_chart
    data="control_transaction_volume"
    x="record_month"
    y="monthly_count"
    y_fmt="num0"
    title="Transaction Timing Triggers -- Volume Control Chart (XmR)"
    chart_options={
        color_palette = ["#006BA4"]
    }
%}
    {% reference_line
        data="control_transaction_volume_limits"
        y="ref_val"
        label="ref_label"
        color="#595959"
        line_options={
            type = "dashed"
            width = 1
        }
    /%}
{% /line_chart %}

```sql control_transaction_dollars
with monthly as (
    select record_month, sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where subcategory = 'Transaction Timing Triggers'
    group by 1
),
with_mr as (
    select record_month, amt,
        abs(amt - lag(amt) over (order by record_month)) as mr
    from monthly
),
stats as (
    select avg(amt) as mean_val, avg(mr) / 1.128 as sigma
    from with_mr
)
select
    m.record_month,
    m.amt as monthly_dollars
from monthly m
order by m.record_month
```

```sql control_transaction_dollars_limits
with monthly as (
    select record_month, sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where subcategory = 'Transaction Timing Triggers'
    group by 1
),
with_mr as (
    select amt, abs(amt - lag(amt) over (order by record_month)) as mr
    from monthly
),
stats as (
    select avg(amt) as mean_val, avg(mr) / 1.128 as sigma
    from with_mr
)
select round(mean_val + 3 * sigma, 0) as ref_val, 'UCL' as ref_label from stats
union all
select round(mean_val, 0), 'Mean' from stats
union all
select greatest(0, round(mean_val - 3 * sigma, 0)), 'LCL' from stats
```

{% line_chart
    data="control_transaction_dollars"
    x="record_month"
    y="monthly_dollars"
    y_fmt="usd0"
    title="Transaction Timing Triggers -- Dollar Volume Control Chart (XmR)"
    chart_options={
        color_palette = ["#FF800E"]
    }
%}
    {% reference_line
        data="control_transaction_dollars_limits"
        y="ref_val"
        label="ref_label"
        color="#595959"
        line_options={
            type = "dashed"
            width = 1
        }
    /%}
{% /line_chart %}

### Owner Pressure Signals

Financial stress short of foreclosure -- liens, judgments, lis pendens. A breakout above the UCL means distress is genuinely increasing, not just a noisy month. This is the early warning for CRA exposure.

```sql control_pressure_volume
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where subcategory = 'Owner Pressure Signals'
    group by 1
),
with_mr as (
    select record_month, cnt,
        abs(cnt - lag(cnt) over (order by record_month)) as mr
    from monthly
),
stats as (
    select avg(cnt) as mean_val, avg(mr) / 1.128 as sigma
    from with_mr
)
select
    m.record_month,
    m.cnt as monthly_count
from monthly m
order by m.record_month
```

```sql control_pressure_volume_limits
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where subcategory = 'Owner Pressure Signals'
    group by 1
),
with_mr as (
    select cnt, abs(cnt - lag(cnt) over (order by record_month)) as mr
    from monthly
),
stats as (
    select avg(cnt) as mean_val, avg(mr) / 1.128 as sigma
    from with_mr
)
select round(mean_val + 3 * sigma, 0) as ref_val, 'UCL' as ref_label from stats
union all
select round(mean_val, 0), 'Mean' from stats
union all
select greatest(0, round(mean_val - 3 * sigma, 0)), 'LCL' from stats
```

{% line_chart
    data="control_pressure_volume"
    x="record_month"
    y="monthly_count"
    y_fmt="num0"
    title="Owner Pressure -- Volume Control Chart (XmR)"
    subtitle="Breakout above UCL = statistically significant increase in financial distress signals"
    chart_options={
        color_palette = ["#006BA4"]
    }
%}
    {% reference_line
        data="control_pressure_volume_limits"
        y="ref_val"
        label="ref_label"
        color="#595959"
        line_options={
            type = "dashed"
            width = 1
        }
    /%}
{% /line_chart %}

```sql control_pressure_dollars
with monthly as (
    select record_month, sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where subcategory = 'Owner Pressure Signals'
    group by 1
),
with_mr as (
    select record_month, amt,
        abs(amt - lag(amt) over (order by record_month)) as mr
    from monthly
),
stats as (
    select avg(amt) as mean_val, avg(mr) / 1.128 as sigma
    from with_mr
)
select
    m.record_month,
    m.amt as monthly_dollars
from monthly m
order by m.record_month
```

```sql control_pressure_dollars_limits
with monthly as (
    select record_month, sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where subcategory = 'Owner Pressure Signals'
    group by 1
),
with_mr as (
    select amt, abs(amt - lag(amt) over (order by record_month)) as mr
    from monthly
),
stats as (
    select avg(amt) as mean_val, avg(mr) / 1.128 as sigma
    from with_mr
)
select round(mean_val + 3 * sigma, 0) as ref_val, 'UCL' as ref_label from stats
union all
select round(mean_val, 0), 'Mean' from stats
union all
select greatest(0, round(mean_val - 3 * sigma, 0)), 'LCL' from stats
```

{% line_chart
    data="control_pressure_dollars"
    x="record_month"
    y="monthly_dollars"
    y_fmt="usd0"
    title="Owner Pressure -- Dollar Volume Control Chart (XmR)"
    chart_options={
        color_palette = ["#FF800E"]
    }
%}
    {% reference_line
        data="control_pressure_dollars_limits"
        y="ref_val"
        label="ref_label"
        color="#595959"
        line_options={
            type = "dashed"
            width = 1
        }
    /%}
{% /line_chart %}

### Equity Unlock Signals

Mortgage releases, satisfactions, reconveyances. Every equity unlock is a borrower who either refinanced with a competitor or paid off entirely. A breakout above the UCL means the competitive landscape has shifted -- you're losing customers at an abnormal rate.

```sql control_equity_volume
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where subcategory = 'Equity Unlock Signals'
    group by 1
),
with_mr as (
    select record_month, cnt,
        abs(cnt - lag(cnt) over (order by record_month)) as mr
    from monthly
),
stats as (
    select avg(cnt) as mean_val, avg(mr) / 1.128 as sigma
    from with_mr
)
select
    m.record_month,
    m.cnt as monthly_count
from monthly m
order by m.record_month
```

```sql control_equity_volume_limits
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where subcategory = 'Equity Unlock Signals'
    group by 1
),
with_mr as (
    select cnt, abs(cnt - lag(cnt) over (order by record_month)) as mr
    from monthly
),
stats as (
    select avg(cnt) as mean_val, avg(mr) / 1.128 as sigma
    from with_mr
)
select round(mean_val + 3 * sigma, 0) as ref_val, 'UCL' as ref_label from stats
union all
select round(mean_val, 0), 'Mean' from stats
union all
select greatest(0, round(mean_val - 3 * sigma, 0)), 'LCL' from stats
```

{% line_chart
    data="control_equity_volume"
    x="record_month"
    y="monthly_count"
    y_fmt="num0"
    title="Equity Unlock -- Volume Control Chart (XmR)"
    subtitle="Breakout above UCL = abnormal rate of borrower departures"
    chart_options={
        color_palette = ["#006BA4"]
    }
%}
    {% reference_line
        data="control_equity_volume_limits"
        y="ref_val"
        label="ref_label"
        color="#595959"
        line_options={
            type = "dashed"
            width = 1
        }
    /%}
{% /line_chart %}

```sql control_equity_dollars
with monthly as (
    select record_month, sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where subcategory = 'Equity Unlock Signals'
    group by 1
),
with_mr as (
    select record_month, amt,
        abs(amt - lag(amt) over (order by record_month)) as mr
    from monthly
),
stats as (
    select avg(amt) as mean_val, avg(mr) / 1.128 as sigma
    from with_mr
)
select
    m.record_month,
    m.amt as monthly_dollars
from monthly m
order by m.record_month
```

```sql control_equity_dollars_limits
with monthly as (
    select record_month, sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where subcategory = 'Equity Unlock Signals'
    group by 1
),
with_mr as (
    select amt, abs(amt - lag(amt) over (order by record_month)) as mr
    from monthly
),
stats as (
    select avg(amt) as mean_val, avg(mr) / 1.128 as sigma
    from with_mr
)
select round(mean_val + 3 * sigma, 0) as ref_val, 'UCL' as ref_label from stats
union all
select round(mean_val, 0), 'Mean' from stats
union all
select greatest(0, round(mean_val - 3 * sigma, 0)), 'LCL' from stats
```

{% line_chart
    data="control_equity_dollars"
    x="record_month"
    y="monthly_dollars"
    y_fmt="usd0"
    title="Equity Unlock -- Dollar Volume Control Chart (XmR)"
    chart_options={
        color_palette = ["#FF800E"]
    }
%}
    {% reference_line
        data="control_equity_dollars_limits"
        y="ref_val"
        label="ref_label"
        color="#595959"
        line_options={
            type = "dashed"
            width = 1
        }
    /%}
{% /line_chart %}

---

## Appendix: Control Chart Method

**XmR (Individuals and Moving Range)** -- Each monthly value is an individual observation. The moving range is the absolute difference between consecutive months. Control limits are calculated as:

- **Center line (Mean):** Average of all monthly values
- **Sigma estimate:** Average moving range / 1.128 (the d2 constant for subgroup size 2)
- **UCL:** Mean + 3 sigma
- **LCL:** Mean - 3 sigma (floored at zero)

XmR is appropriate here because each month is a single observation with no subgrouping. A point outside the control limits has less than a 0.27% chance of occurring under stable conditions -- it represents a genuine process shift, not random variation.

The 3-year lookback window from `silver.monthly_signal_velocity` provides 36 data points per subcategory, which is sufficient for stable limit estimation. If Guam's recording patterns show overdispersion (common in large-count data), a Laney u' adjustment can be applied to widen the limits and reduce false signals.