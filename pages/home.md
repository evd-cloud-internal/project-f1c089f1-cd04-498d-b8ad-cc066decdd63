---
name: Home
assetId: c18a30f5-1d9f-4e71-91d9-e5161bcc6957
type: page
---

# Executive Signals Overview

Every property on Guam tells a story through its recorded instruments. A mortgage filed today is a lending relationship born. A lien recorded tomorrow is financial pressure building. A satisfaction of mortgage next week is a borrower who just freed their collateral--and is now either a competitor's customer or your next HELOC opportunity.

RealFACTS reads these stories at scale. The platform processes recorded instruments from the Guam Department of Land Management, classifying each one into signal families that answer five questions a potential lending partner cares about:

- Who will transact next?
- Will this deal close?
- What's the capital position?
- Where is the market moving?
- Where are the compliance risks?

This page is the executive nerve center. It starts with what needs your attention right now, then unfolds category by category--showing you not just what happened, but whether the change is real or just noise.

**How to read this page:** Prose sections explain what each signal family contains, how the charts are constructed, and what any hidden SQL filtering or bucketing does. The charts and tables tell the story--the prose does not. Each section ends with a **So what?** block: starter questions to guide investigation, framed as possibilities rather than conclusions. If a query excludes certain data, applies an offset, or uses a time window, it is called out in the prose and commented in the SQL.

---

## Platform Snapshot

```sql platform_snapshot
-- Live counts from audit and silver layers. No hardcoded numbers.
-- audit.signal_category_coverage: instrument-level classification status
-- audit.data_freshness: pipeline currency
-- silver.property_signal_summary: distinct properties with bridge links
-- bronze.mortgages: active mortgage portfolio
SELECT
    (SELECT SUM(instrument_count) FROM audit_signal_category_coverage) AS total_instruments,
    (SELECT SUM(instrument_count) FROM audit_signal_category_coverage WHERE mapping_status = 'MAPPED') AS classified_instruments,
    (SELECT ROUND(100.0 * SUM(instrument_count) FILTER (WHERE mapping_status = 'MAPPED') / SUM(instrument_count), 1) FROM audit_signal_category_coverage) AS pct_classified,
    (SELECT COUNT(DISTINCT legal_unit_id) FROM silver_property_signal_summary) AS distinct_properties,
    (SELECT COUNT(*) FROM bronze_mortgages WHERE is_likely_active) AS active_mortgages,
    (SELECT most_recent_record FROM audit_data_freshness) AS data_through,
    (SELECT days_since_last_record FROM audit_data_freshness) AS days_since_latest
```

{% row %}
    {% big_value
        data="platform_snapshot"
        value="total_instruments"
        title="Recorded Instruments"
        fmt="num0"
    /%}
    {% big_value
        data="platform_snapshot"
        value="pct_classified"
        title="Classification Coverage"
        fmt="pct1"
    /%}
    {% big_value
        data="platform_snapshot"
        value="distinct_properties"
        title="Linked Properties"
        fmt="num0"
    /%}
    {% big_value
        data="platform_snapshot"
        value="active_mortgages"
        title="Active Mortgages"
        fmt="num0"
    /%}
{% /row %}

These numbers are live from the audit and silver layers. Classification coverage reflects the percentage of all recorded instruments mapped to a signal category via `dim.signal_category`. Linked properties counts distinct legal units with at least one bridge-resolved instrument. Active mortgages reflects the `is_likely_active` flag in `bronze.mortgages`, which uses mortgage term assumptions to estimate whether each mortgage has been paid off.

---

## Attention Now: Signals Outside Normal Range

This table scans every signal subcategory and flags only the ones where the evaluated month fell outside 3-sigma statistical control limits (XmR method--see Appendix B).

**How to read:** If nothing appears, no subcategory breached its control limits. If rows appear, the volume_flag and dollar_flag columns indicate whether the value was above the upper control limit or below the lower control limit. The `evaluation_month` column shows which month was tested.

**Hidden filter:** This query evaluates the **second-most-recent month** in the dataset (offset 1), not the most recent. The most recent month may contain partial recording data. If you need to evaluate the most recent month regardless of completeness, change `offset 1` to `offset 0` in the SQL.

**Exclusion:** Metadata category is excluded (administrative filings with no signal value).

```sql out_of_control_summary
with monthly as (
    select
        category,
        subcategory,
        record_month,
        sum(current_month_count) as cnt,
        sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    -- Metadata excluded: administrative filings with no signal value
    where category != 'Metadata'
    group by 1, 2, 3
),
with_mr as (
    select
        category,
        subcategory,
        record_month,
        cnt,
        amt,
        abs(cnt - lag(cnt) over (partition by subcategory order by record_month)) as mr_cnt,
        abs(amt - lag(amt) over (partition by subcategory order by record_month)) as mr_amt
    from monthly
),
stats as (
    select
        category,
        subcategory,
        avg(cnt) as mean_cnt,
        avg(mr_cnt) / 1.128 as sigma_cnt,
        avg(amt) as mean_amt,
        avg(mr_amt) / 1.128 as sigma_amt
    from with_mr
    group by 1, 2
),
-- OFFSET 1: Evaluate the second-most-recent month, not the latest.
-- Reason: the most recent month may have partial recording data.
-- Change to offset 0 to evaluate the latest month regardless.
eval_month as (
    select distinct record_month
    from gold_signal_velocity_trends
    order by record_month desc
    limit 1 offset 1
),
latest as (
    select
        m.category,
        m.subcategory,
        m.record_month as evaluation_month,
        m.cnt as current_count,
        m.amt as current_dollars,
        s.mean_cnt,
        round(s.mean_cnt + 3 * s.sigma_cnt, 0) as ucl_count,
        greatest(0, round(s.mean_cnt - 3 * s.sigma_cnt, 0)) as lcl_count,
        s.mean_amt,
        round(s.mean_amt + 3 * s.sigma_amt, 0) as ucl_dollars,
        greatest(0, round(s.mean_amt - 3 * s.sigma_amt, 0)) as lcl_dollars
    from monthly m
    inner join stats s on m.subcategory = s.subcategory
    where m.record_month = (select record_month from eval_month)
)
select
    evaluation_month,
    subcategory,
    current_count,
    round(mean_cnt, 0) as mean_count,
    ucl_count,
    lcl_count,
    case
        when current_count > ucl_count then 'ABOVE UCL'
        when current_count < lcl_count then 'BELOW LCL'
        else null
    end as volume_flag,
    current_dollars,
    round(mean_amt, 0) as mean_dollars,
    ucl_dollars,
    lcl_dollars,
    case
        when current_dollars > ucl_dollars then 'ABOVE UCL'
        when current_dollars < lcl_dollars then 'BELOW LCL'
        else null
    end as dollar_flag
from latest
where current_count > ucl_count
   or current_count < lcl_count
   or current_dollars > ucl_dollars
   or current_dollars < lcl_dollars
order by subcategory
```

{% table data="out_of_control_summary" page_size=20 /%}

**So what?** For each flagged subcategory: Is this a one-month anomaly or part of a trend visible in the control charts below? Could an external event (policy change, seasonal pattern, recording backlog) explain it? Does the dollar flag tell a different story than the volume flag--e.g., fewer instruments but larger dollar amounts? Which lending operations (origination, retention, counseling, compliance) should be alerted?

---

## Latest Month at a Glance

**How to read:** Each card shows the most recent month's value with a comparison delta against the prior month. The comparison arrow indicates direction and magnitude of change.

**Hidden filter:** Unlike the Attention Now table above, this section uses the **most recent month** in the dataset (no offset). This means the current month may contain partial data if recording is still in progress. Metadata category is excluded from Total Signals.

```sql latest_month_kpis
-- Anchor: the two most recent months in the dataset (no offset).
-- NOTE: Unlike out_of_control_summary, this uses the LATEST month,
-- which may be partial if recording is still in progress.
-- curr = latest month, prev = month before that.
with anchor as (
    select distinct record_month
    from gold_signal_velocity_trends
    order by record_month desc
    limit 2
),
current_month as (select max(record_month) as mo from anchor),
prior_month as (select min(record_month) as mo from anchor),
curr as (
    select
        -- Metadata excluded from total: administrative filings with no signal value
        sum(current_month_count) filter (where category != 'Metadata') as total_signals,
        sum(current_month_count) filter (where category = 'Transaction Propensity Intelligence') as transaction_propensity,
        sum(current_month_count) filter (where subcategory = 'Owner Pressure Signals') as owner_pressure,
        sum(current_month_count) filter (where subcategory = 'Equity Unlock Signals') as equity_unlocks,
        sum(current_month_count) filter (where subcategory = 'Forced Sale Signals') as forced_sales,
        sum(current_month_count) filter (where subcategory = 'Blocker Clearance Signals') as blockers_cleared,
        sum(current_month_amount) filter (where category = 'Capital & Leverage Intelligence') as capital_leverage_volume
    from gold_signal_velocity_trends
    where record_month = (select mo from current_month)
),
prev as (
    select
        sum(current_month_count) filter (where category != 'Metadata') as total_signals,
        sum(current_month_count) filter (where category = 'Transaction Propensity Intelligence') as transaction_propensity,
        sum(current_month_count) filter (where subcategory = 'Owner Pressure Signals') as owner_pressure,
        sum(current_month_count) filter (where subcategory = 'Equity Unlock Signals') as equity_unlocks,
        sum(current_month_count) filter (where subcategory = 'Forced Sale Signals') as forced_sales,
        sum(current_month_count) filter (where subcategory = 'Blocker Clearance Signals') as blockers_cleared,
        sum(current_month_amount) filter (where category = 'Capital & Leverage Intelligence') as capital_leverage_volume
    from gold_signal_velocity_trends
    where record_month = (select mo from prior_month)
)
select
    curr.total_signals,
    prev.total_signals as prior_total_signals,
    curr.transaction_propensity,
    prev.transaction_propensity as prior_transaction_propensity,
    curr.owner_pressure,
    prev.owner_pressure as prior_owner_pressure,
    curr.equity_unlocks,
    prev.equity_unlocks as prior_equity_unlocks,
    curr.forced_sales,
    prev.forced_sales as prior_forced_sales,
    curr.blockers_cleared,
    prev.blockers_cleared as prior_blockers_cleared,
    curr.capital_leverage_volume,
    prev.capital_leverage_volume as prior_capital_leverage_volume
from curr cross join prev
```

{% row %}
    {% big_value
        data="latest_month_kpis"
        value="total_signals"
        title="Total Signals"
        fmt="num0"
        comparison="prior_total_signals"
        comparison_title="vs prior month"
    /%}
    {% big_value
        data="latest_month_kpis"
        value="transaction_propensity"
        title="Transaction Propensity"
        fmt="num0"
        comparison="prior_transaction_propensity"
        comparison_title="vs prior month"
    /%}
    {% big_value
        data="latest_month_kpis"
        value="capital_leverage_volume"
        title="Capital & Leverage Volume"
        fmt="usd0"
        comparison="prior_capital_leverage_volume"
        comparison_title="vs prior month"
    /%}
{% /row %}

{% row %}
    {% big_value
        data="latest_month_kpis"
        value="equity_unlocks"
        title="Equity Unlocks"
        fmt="num0"
        comparison="prior_equity_unlocks"
        comparison_title="vs prior month"
    /%}
    {% big_value
        data="latest_month_kpis"
        value="owner_pressure"
        title="Owner Pressure"
        fmt="num0"
        comparison="prior_owner_pressure"
        comparison_title="vs prior month"
    /%}
    {% big_value
        data="latest_month_kpis"
        value="forced_sales"
        title="Forced Sales"
        fmt="num0"
        comparison="prior_forced_sales"
        comparison_title="vs prior month"
    /%}
    {% big_value
        data="latest_month_kpis"
        value="blockers_cleared"
        title="Blockers Cleared"
        fmt="num0"
        comparison="prior_blockers_cleared"
        comparison_title="vs prior month"
    /%}
{% /row %}

### Month-over-Month by Subcategory

**How to read:** Sorted by largest absolute count change. The pct_change column shows the percentage shift relative to the prior month.

**Hidden filter:** Same two-month anchor as the KPIs above (most recent month, no offset). Metadata excluded.

```sql period_comparison
-- Same anchor as latest_month_kpis: most recent two months, no offset.
with anchor as (
    select distinct record_month
    from gold_signal_velocity_trends
    order by record_month desc
    limit 2
),
current_month as (select max(record_month) as mo from anchor),
prior_month as (select min(record_month) as mo from anchor)
select
    c.subcategory,
    c.cnt as current_count,
    p.cnt as prior_count,
    c.cnt - p.cnt as count_delta,
    round(100.0 * (c.cnt - p.cnt) / nullif(p.cnt, 0), 1) as pct_change,
    c.amt as current_dollars,
    p.amt as prior_dollars,
    c.amt - p.amt as dollar_delta
from (
    select subcategory, sum(current_month_count) as cnt, sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where record_month = (select mo from current_month)
      and category != 'Metadata'  -- Metadata excluded
    group by 1
) c
inner join (
    select subcategory, sum(current_month_count) as cnt, sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where record_month = (select mo from prior_month)
      and category != 'Metadata'  -- Metadata excluded
    group by 1
) p on c.subcategory = p.subcategory
order by abs(c.cnt - p.cnt) desc
```

{% table data="period_comparison" page_size=20 /%}

**So what?** Which subcategories show the largest swings? Are the biggest movers high-weight signals (Capital & Leverage, Forced Sales) or lower-weight administrative categories? Does the dollar delta tell a different story than the count delta--e.g., fewer instruments but larger individual amounts? Could any large swing be explained by a recording backlog clearing?

---

## Divergence Monitor: Paired Signals That Tell a Story

**How to read:** These charts overlay two related subcategories on the same axis. The diagnostic value is in whether the lines move together or apart. Lighter lines show 3-month and 6-month moving averages to reveal the underlying trend beneath monthly noise.

**Time window:** Full 3-year window from the underlying gold view (no additional filtering).

### Equity Unlocks vs Owner Pressure

**What's in each series:** Equity Unlock Signals represent collateral being freed (satisfactions, releases, discharges). Owner Pressure Signals represent financial stress building (liens, judgments, lis pendens).

**How to read the pair:** The relationship between these two lines is the diagnostic, not either line alone. Same direction = reinforcing. Opposite direction = one signal resolving while the other builds.

```sql divergence_equity_pressure
-- Full 3-year window from gold view. No additional time filter.
with base as (
    select
        record_month,
        subcategory,
        sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where subcategory in ('Equity Unlock Signals', 'Owner Pressure Signals')
    group by 1, 2
),
with_ma as (
    select
        record_month,
        subcategory,
        cnt,
        round(avg(cnt) over (partition by subcategory order by record_month rows between 2 preceding and current row), 0) as ma_3,
        round(avg(cnt) over (partition by subcategory order by record_month rows between 5 preceding and current row), 0) as ma_6
    from base
)
select record_month, subcategory as series, cnt as monthly_count from with_ma
union all select record_month, subcategory || ' (3-Mo MA)', ma_3 from with_ma
union all select record_month, subcategory || ' (6-Mo MA)', ma_6 from with_ma
order by 1, 2
```

{% line_chart
    data="divergence_equity_pressure"
    x="record_month"
    y="monthly_count"
    series="series"
    series_order=["Equity Unlock Signals", "Owner Pressure Signals", "Equity Unlock Signals (3-Mo MA)", "Owner Pressure Signals (3-Mo MA)", "Equity Unlock Signals (6-Mo MA)", "Owner Pressure Signals (6-Mo MA)"]
    y_fmt="num0"
    title="Equity Unlocks vs Owner Pressure -- Monthly Volume"
    subtitle="Same direction = reinforcing. Opposite direction = diagnostic. Lighter lines = smoothed trend."
    chart_options={
        series_colors = {
            "Equity Unlock Signals" = "#2CA02C"
            "Equity Unlock Signals (3-Mo MA)" = "#98DF8A"
            "Equity Unlock Signals (6-Mo MA)" = "#B5E0B5"
            "Owner Pressure Signals" = "#E17000"
            "Owner Pressure Signals (3-Mo MA)" = "#F2C99B"
            "Owner Pressure Signals (6-Mo MA)" = "#F5DFC5"
        }
    }
/%}

**So what?** Are equity unlocks rising while owner pressure falls--could that indicate borrowers paying off voluntarily in a healthy market? Are both rising together--might borrowers be liquidating under pressure? Is pressure rising while unlocks stay flat--could stress be building with no relief valve? What external factors (interest rate changes, seasonal patterns, policy shifts) might explain the current relationship?

### Capital & Leverage vs Forced Sales

**What's in each series:** Capital & Leverage represents new lending activity (mortgages, modifications, subordinations--all subcategories rolled into one line). Forced Sale Signals represent involuntary property transfers (foreclosure deeds, tax deeds, trustee deeds--single subcategory).

**How to read the pair:** The relationship between origination volume and forced sale volume is the diagnostic. Convergence or divergence between these lines is the signal.

```sql divergence_capital_forced
-- Full 3-year window. Capital & Leverage aggregated to category level.
with base as (
    select
        record_month,
        -- Bucketing: all Capital & Leverage subcategories rolled into one line
        case
            when category = 'Capital & Leverage Intelligence' then 'Capital & Leverage'
            else subcategory
        end as signal_name,
        sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where category = 'Capital & Leverage Intelligence'
       or subcategory = 'Forced Sale Signals'
    group by 1, 2
),
with_ma as (
    select
        record_month,
        signal_name,
        cnt,
        round(avg(cnt) over (partition by signal_name order by record_month rows between 2 preceding and current row), 0) as ma_3,
        round(avg(cnt) over (partition by signal_name order by record_month rows between 5 preceding and current row), 0) as ma_6
    from base
)
select record_month, signal_name as series, cnt as monthly_count from with_ma
union all select record_month, signal_name || ' (3-Mo MA)', ma_3 from with_ma
union all select record_month, signal_name || ' (6-Mo MA)', ma_6 from with_ma
order by 1, 2
```

{% line_chart
    data="divergence_capital_forced"
    x="record_month"
    y="monthly_count"
    series="series"
    series_order=["Capital & Leverage", "Forced Sale Signals", "Capital & Leverage (3-Mo MA)", "Forced Sale Signals (3-Mo MA)", "Capital & Leverage (6-Mo MA)", "Forced Sale Signals (6-Mo MA)"]
    y_fmt="num0"
    title="Capital & Leverage vs Forced Sales -- Monthly Volume"
    subtitle="Origination volume vs involuntary transfers. Convergence or divergence is the signal."
    chart_options={
        series_colors = {
            "Capital & Leverage" = "#006BA4"
            "Capital & Leverage (3-Mo MA)" = "#5F9ED1"
            "Capital & Leverage (6-Mo MA)" = "#A2C8EC"
            "Forced Sale Signals" = "#C44601"
            "Forced Sale Signals (3-Mo MA)" = "#E8A77C"
            "Forced Sale Signals (6-Mo MA)" = "#F2CDB8"
        }
    }
/%}

**So what?** Are originations rising while forced sales stay flat or decline--could this suggest a healthy and expanding market? Are originations falling while forced sales rise--might the credit cycle be turning? Are both falling--could the market be contracting overall? How does the current relationship compare to the 3-year trend visible in the moving averages?

---

## Market Pulse: All Signal Categories

**How to read:** Five signal families (Metadata excluded), compared month-over-month with a 6-month moving average for trend context. Sorted by current count descending.

**Hidden filter:** Same two-month anchor as KPIs (most recent month, no offset). Metadata excluded.

```sql market_pulse_summary
-- Same two-month anchor pattern as latest_month_kpis. Metadata excluded.
with monthly as (
    select
        record_month,
        category,
        sum(current_month_count) as cnt,
        sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where category != 'Metadata'  -- Metadata excluded
    group by 1, 2
),
with_ma as (
    select
        record_month,
        category,
        cnt,
        amt,
        round(avg(cnt) over (partition by category order by record_month rows between 5 preceding and current row), 0) as ma_6_count,
        round(avg(amt) over (partition by category order by record_month rows between 5 preceding and current row), 0) as ma_6_dollars
    from monthly
),
anchor as (
    select distinct record_month
    from monthly
    order by record_month desc
    limit 2
),
current_month as (select max(record_month) as mo from anchor),
prior_month as (select min(record_month) as mo from anchor)
select
    c.category,
    c.cnt as current_count,
    p.cnt as prior_count,
    c.cnt - p.cnt as count_delta,
    c.ma_6_count as six_mo_avg_count,
    c.amt as current_dollars,
    p.amt as prior_dollars,
    c.amt - p.amt as dollar_delta,
    c.ma_6_dollars as six_mo_avg_dollars
from with_ma c
inner join with_ma p on c.category = p.category
cross join current_month cm
cross join prior_month pm
where c.record_month = cm.mo
  and p.record_month = pm.mo
order by c.cnt desc
```

{% table data="market_pulse_summary" page_size=20 /%}

**So what?** Which categories are above or below their 6-month average? Is the delta concentrated in one category or spread across the board? Are dollar deltas moving in the same direction as count deltas, or is the average deal size shifting? Which category movements, if sustained, would most affect lending operations?

---

## YTD Pace vs Prior Year

**How to read:** Compares the current year's cumulative volume (through the latest available month) against the same calendar months in the prior year. The pct_change column shows year-over-year growth or contraction.

**Hidden filter:** The prior year comparison is limited to the same months available in the current year (apples-to-apples). Metadata excluded.

```sql ytd_pace
-- YTD comparison: current year through latest month vs same months prior year.
-- Prior year is filtered to month(record_month) <= latest month in current year
-- to ensure apples-to-apples comparison.
with latest_month as (
    select max(month(record_month)) as mo
    from gold_signal_velocity_trends
    where record_month >= date_trunc('year', (select max(record_month) from gold_signal_velocity_trends))
),
current_ytd as (
    select
        category,
        sum(current_month_count)::double as ytd_count,
        sum(current_month_amount)::double as ytd_dollars
    from gold_signal_velocity_trends
    where category != 'Metadata'  -- Metadata excluded
      and record_month >= date_trunc('year', (select max(record_month) from gold_signal_velocity_trends))
    group by 1
),
prior_ytd as (
    select
        category,
        sum(current_month_count)::double as ytd_count,
        sum(current_month_amount)::double as ytd_dollars
    from gold_signal_velocity_trends
    where category != 'Metadata'  -- Metadata excluded
      -- Prior year: same calendar year window, offset by 1 year
      and record_month >= date_trunc('year', (select max(record_month) from gold_signal_velocity_trends) - interval '1' year)
      and record_month < date_trunc('year', (select max(record_month) from gold_signal_velocity_trends))
      -- Apples-to-apples: only include months up to the latest month available this year
      and month(record_month) <= (select mo from latest_month)
    group by 1
)
select
    c.category,
    c.ytd_count as current_ytd,
    p.ytd_count as prior_year_same_period,
    c.ytd_count - p.ytd_count as delta,
    round(100.0 * (c.ytd_count - p.ytd_count) / nullif(p.ytd_count, 0), 1) as pct_change,
    c.ytd_dollars as current_ytd_dollars,
    p.ytd_dollars as prior_ytd_dollars,
    round(100.0 * (c.ytd_dollars - p.ytd_dollars) / nullif(p.ytd_dollars, 0), 1) as dollar_pct_change
from current_ytd c
left join prior_ytd p on c.category = p.category
order by c.ytd_count desc
```

{% table data="ytd_pace" page_size=20 /%}

**So what?** Which categories are pacing ahead of last year, and which are behind? Is the year-over-year change consistent with the monthly trends visible in the control charts, or is one quarter pulling the number? Are dollar changes proportional to count changes, or is the average instrument value shifting?

---

## Signal Concentration: Where Does the Volume Live?

**How to read:** Pareto analysis ranks subcategories by volume (or dollar exposure) over the trailing 12 months. The cumulative_pct column shows how much of total activity the top subcategories account for. The in_80 flag marks subcategories in the top 80%.

**Time window:** Trailing 12 months from the latest month in the dataset. Metadata excluded.

### Volume Pareto -- Last 12 Months

```sql pareto_volume
-- Trailing 12 months from the latest month. Metadata excluded.
with subcategory_totals as (
    select
        subcategory,
        sum(current_month_count) as total_count
    from gold_signal_velocity_trends
    where category != 'Metadata'  -- Metadata excluded
      -- Trailing 12-month window
      and record_month >= (
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
        sum(total_count) over () as grand_total,
        row_number() over (order by total_count desc) as rn
    from subcategory_totals
)
select
    lpad(rn::varchar, 2, '0') as rank,
    subcategory,
    total_count,
    round(100.0 * total_count / grand_total, 1) as pct_of_total,
    round(100.0 * running_total / grand_total, 1) as cumulative_pct,
    case when running_total - total_count < 0.8 * grand_total then 'Top 80%' else '' end as in_80
from ranked
order by rn
```

{% table data="pareto_volume" page_size=20 /%}

### Dollar Pareto -- Last 12 Months

```sql pareto_dollars
-- Trailing 12 months from the latest month. Metadata excluded.
with subcategory_totals as (
    select
        subcategory,
        sum(current_month_amount) as total_amount
    from gold_signal_velocity_trends
    where category != 'Metadata'  -- Metadata excluded
      -- Trailing 12-month window
      and record_month >= (
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
        sum(total_amount) over () as grand_total,
        row_number() over (order by total_amount desc) as rn
    from subcategory_totals
)
select
    lpad(rn::varchar, 2, '0') as rank,
    subcategory,
    total_amount,
    round(100.0 * total_amount / nullif(grand_total, 0), 1) as pct_of_total,
    round(100.0 * running_total / nullif(grand_total, 0), 1) as cumulative_pct,
    case when running_total - total_amount < 0.8 * grand_total then 'Top 80%' else '' end as in_80
from ranked
order by rn
```

{% table data="pareto_dollars" page_size=20 /%}

**So what?** How many subcategories drive 80% of volume? Is dollar concentration tighter or wider than volume concentration--i.e., are a few high-dollar subcategories dominating the dollar Pareto while volume is more spread out? Are the top-ranked subcategories the ones with the highest signal and revenue weights, or are lower-weight categories consuming disproportionate attention?

---

## Signal Family: Capital & Leverage Intelligence

**The question this answers:** "What's the capital position, and what financial actions are optimal now?"

**What's in this family:** Mortgages, modifications, subordination agreements, bonds, promissory notes, assumptions, and assignment variants tracking servicing transfers. Classification weights: signal=10, distress=2, revenue=10. Low distress (restructuring, not default), maximum signal and revenue. See Appendix A for the full instrument type list as defined in `dim.signal_category`.

**How to read:** The volume chart shows instrument count per month; the dollar chart shows total dollar value per month. Points outside the dashed control limits represent statistically significant shifts (see Appendix B). Moving averages smooth monthly noise.

```sql control_capital_leverage_volume
-- Full 3-year window from gold view. No additional time filter.
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where category = 'Capital & Leverage Intelligence'
    group by 1
),
with_ma as (
    select record_month, cnt,
        round(avg(cnt) over (order by record_month rows between 2 preceding and current row), 0) as ma_3,
        round(avg(cnt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, cnt as value from with_ma
union all select record_month, '3-Mo MA', ma_3 from with_ma
union all select record_month, '6-Mo MA', ma_6 from with_ma
order by 1, 2
```

```sql control_capital_leverage_volume_limits
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where category = 'Capital & Leverage Intelligence'
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
union all select round(mean_val, 0), 'Mean' from stats
union all select greatest(0, round(mean_val - 3 * sigma, 0)), 'LCL' from stats
```

{% line_chart
    data="control_capital_leverage_volume"
    x="record_month"
    y="value"
    series="series"
    series_order=["Monthly", "3-Mo MA", "6-Mo MA"]
    y_fmt="num0"
    title="Capital & Leverage Intelligence -- Volume Control Chart"
    subtitle="3-year window. Dashed lines = UCL / Mean / LCL (3-sigma XmR limits)."
    chart_options={
        series_colors = {
            "Monthly" = "#006BA4"
            "3-Mo MA" = "#5F9ED1"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_capital_leverage_volume_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}

```sql control_capital_leverage_dollars
-- Full 3-year window from gold view. No additional time filter.
with monthly as (
    select record_month, sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where category = 'Capital & Leverage Intelligence'
    group by 1
),
with_ma as (
    select record_month, amt,
        round(avg(amt) over (order by record_month rows between 2 preceding and current row), 0) as ma_3,
        round(avg(amt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, amt as value from with_ma
union all select record_month, '3-Mo MA', ma_3 from with_ma
union all select record_month, '6-Mo MA', ma_6 from with_ma
order by 1, 2
```

```sql control_capital_leverage_dollars_limits
with monthly as (
    select record_month, sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where category = 'Capital & Leverage Intelligence'
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
union all select round(mean_val, 0), 'Mean' from stats
union all select greatest(0, round(mean_val - 3 * sigma, 0)), 'LCL' from stats
```

{% line_chart
    data="control_capital_leverage_dollars"
    x="record_month"
    y="value"
    series="series"
    series_order=["Monthly", "3-Mo MA", "6-Mo MA"]
    y_fmt="usd0"
    title="Capital & Leverage Intelligence -- Dollar Volume Control Chart"
    subtitle="3-year window. Dashed lines = UCL / Mean / LCL (3-sigma XmR limits)."
    chart_options={
        series_colors = {
            "Monthly" = "#FF800E"
            "3-Mo MA" = "#FFBC79"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_capital_leverage_dollars_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}

**So what?** Is lending activity rising, falling, or stable relative to the 3-year baseline? Could a sustained drop below the lower control limit indicate genuine market contraction rather than a seasonal dip? If volume is rising but dollar volume is flat, could average deal sizes be shrinking--and what might that mean for portfolio composition? Does the current trend align with the interest rate environment? Could a spike signal a refinance wave worth investigating in the rate-gap analysis?

---

## Signal Family: Transaction Propensity Intelligence

**The question this answers:** "Who will transact next, and why?"

**What's in this family:** The largest signal family, spanning 7 subcategories that range from healthy market activity (Transaction Timing Triggers) through escalating financial stress (Owner Pressure, Forced Sales) to resolution (Blocker Clearance, Equity Unlock). Life and Legal Triggers sit outside this spectrum as external shocks (death, divorce, probate) that may force property decisions regardless of market conditions. See Appendix A for subcategory weights.

### Category-Level Overview

**How to read:** This is the aggregate of all 7 subcategories. Use the subcategory charts below to decompose any movement visible here.

```sql control_txn_propensity_volume
-- Full 3-year window. All Transaction Propensity subcategories combined.
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where category = 'Transaction Propensity Intelligence'
    group by 1
),
with_ma as (
    select record_month, cnt,
        round(avg(cnt) over (order by record_month rows between 2 preceding and current row), 0) as ma_3,
        round(avg(cnt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, cnt as value from with_ma
union all select record_month, '3-Mo MA', ma_3 from with_ma
union all select record_month, '6-Mo MA', ma_6 from with_ma
order by 1, 2
```

```sql control_txn_propensity_volume_limits
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where category = 'Transaction Propensity Intelligence'
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
union all select round(mean_val, 0), 'Mean' from stats
union all select greatest(0, round(mean_val - 3 * sigma, 0)), 'LCL' from stats
```

{% line_chart
    data="control_txn_propensity_volume"
    x="record_month"
    y="value"
    series="series"
    series_order=["Monthly", "3-Mo MA", "6-Mo MA"]
    y_fmt="num0"
    title="Transaction Propensity Intelligence -- Volume Control Chart"
    subtitle="All 7 subcategories combined. 3-year window."
    chart_options={
        series_colors = {
            "Monthly" = "#006BA4"
            "3-Mo MA" = "#5F9ED1"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_txn_propensity_volume_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}

```sql control_txn_propensity_dollars
-- Full 3-year window.
with monthly as (
    select record_month, sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where category = 'Transaction Propensity Intelligence'
    group by 1
),
with_ma as (
    select record_month, amt,
        round(avg(amt) over (order by record_month rows between 2 preceding and current row), 0) as ma_3,
        round(avg(amt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, amt as value from with_ma
union all select record_month, '3-Mo MA', ma_3 from with_ma
union all select record_month, '6-Mo MA', ma_6 from with_ma
order by 1, 2
```

```sql control_txn_propensity_dollars_limits
with monthly as (
    select record_month, sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where category = 'Transaction Propensity Intelligence'
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
union all select round(mean_val, 0), 'Mean' from stats
union all select greatest(0, round(mean_val - 3 * sigma, 0)), 'LCL' from stats
```

{% line_chart
    data="control_txn_propensity_dollars"
    x="record_month"
    y="value"
    series="series"
    series_order=["Monthly", "3-Mo MA", "6-Mo MA"]
    y_fmt="usd0"
    title="Transaction Propensity Intelligence -- Dollar Volume Control Chart"
    subtitle="All 7 subcategories combined. 3-year window."
    chart_options={
        series_colors = {
            "Monthly" = "#FF800E"
            "3-Mo MA" = "#FFBC79"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_txn_propensity_dollars_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}

**So what?** Is the aggregate moving because one subcategory is spiking, or is the shift broad-based? Check the subcategory charts below to decompose. Is the dollar chart telling the same story as the volume chart?


### Transaction Timing Triggers

**What's in this subcategory:** Deeds, conveyances, leases, contracts, options, maps, and bills of sale. Active market transactions. Weights: signal=8, distress=1, revenue=8.

**How to read:** These represent the deposits pipeline and new relationship opportunities. A deed transfer or notice of sale may be a leading indicator of subsequent transaction within the property's ownership lifecycle.

```sql control_timing_volume
-- Full 3-year window.
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where subcategory = 'Transaction Timing Triggers'
    group by 1
),
with_ma as (
    select record_month, cnt,
        round(avg(cnt) over (order by record_month rows between 2 preceding and current row), 0) as ma_3,
        round(avg(cnt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, cnt as value from with_ma
union all select record_month, '3-Mo MA', ma_3 from with_ma
union all select record_month, '6-Mo MA', ma_6 from with_ma
order by 1, 2
```

```sql control_timing_volume_limits
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
union all select round(mean_val, 0), 'Mean' from stats
union all select greatest(0, round(mean_val - 3 * sigma, 0)), 'LCL' from stats
```

{% line_chart
    data="control_timing_volume"
    x="record_month"
    y="value"
    series="series"
    series_order=["Monthly", "3-Mo MA", "6-Mo MA"]
    y_fmt="num0"
    title="Transaction Timing Triggers -- Volume Control Chart"
    subtitle="3-year window. Dashed lines = 3-sigma XmR limits."
    chart_options={
        series_colors = {
            "Monthly" = "#006BA4"
            "3-Mo MA" = "#5F9ED1"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_timing_volume_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}

```sql control_timing_dollars
-- Full 3-year window.
with monthly as (
    select record_month, sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where subcategory = 'Transaction Timing Triggers'
    group by 1
),
with_ma as (
    select record_month, amt,
        round(avg(amt) over (order by record_month rows between 2 preceding and current row), 0) as ma_3,
        round(avg(amt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, amt as value from with_ma
union all select record_month, '3-Mo MA', ma_3 from with_ma
union all select record_month, '6-Mo MA', ma_6 from with_ma
order by 1, 2
```

```sql control_timing_dollars_limits
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
union all select round(mean_val, 0), 'Mean' from stats
union all select greatest(0, round(mean_val - 3 * sigma, 0)), 'LCL' from stats
```

{% line_chart
    data="control_timing_dollars"
    x="record_month"
    y="value"
    series="series"
    series_order=["Monthly", "3-Mo MA", "6-Mo MA"]
    y_fmt="usd0"
    title="Transaction Timing Triggers -- Dollar Volume Control Chart"
    subtitle="3-year window. Dashed lines = 3-sigma XmR limits."
    chart_options={
        series_colors = {
            "Monthly" = "#FF800E"
            "3-Mo MA" = "#FFBC79"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_timing_dollars_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}

**So what?** Is the transaction pipeline growing or shrinking? Are recent deed transfers concentrated in certain property types or geographies that could indicate emerging market pockets? Could a rising trend here suggest an expanding deposits pipeline or new-relationship outreach window?


### Equity Unlock Signals

**What's in this subcategory:** Mortgage releases, satisfactions, discharges, and judgment cancellations. Weights: signal=9, distress=1, revenue=9.

**How to read:** Each equity unlock is a borrower who freed their collateral. The reason--payoff, refinance with another lender, or property sale--determines the opportunity type and optimal response.

```sql control_equity_volume
-- Full 3-year window.
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where subcategory = 'Equity Unlock Signals'
    group by 1
),
with_ma as (
    select record_month, cnt,
        round(avg(cnt) over (order by record_month rows between 2 preceding and current row), 0) as ma_3,
        round(avg(cnt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, cnt as value from with_ma
union all select record_month, '3-Mo MA', ma_3 from with_ma
union all select record_month, '6-Mo MA', ma_6 from with_ma
order by 1, 2
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
union all select round(mean_val, 0), 'Mean' from stats
union all select greatest(0, round(mean_val - 3 * sigma, 0)), 'LCL' from stats
```

{% line_chart
    data="control_equity_volume"
    x="record_month"
    y="value"
    series="series"
    series_order=["Monthly", "3-Mo MA", "6-Mo MA"]
    y_fmt="num0"
    title="Equity Unlock Signals -- Volume Control Chart"
    subtitle="3-year window. Dashed lines = 3-sigma XmR limits."
    chart_options={
        series_colors = {
            "Monthly" = "#006BA4"
            "3-Mo MA" = "#5F9ED1"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_equity_volume_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}

```sql control_equity_dollars
-- Full 3-year window.
with monthly as (
    select record_month, sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where subcategory = 'Equity Unlock Signals'
    group by 1
),
with_ma as (
    select record_month, amt,
        round(avg(amt) over (order by record_month rows between 2 preceding and current row), 0) as ma_3,
        round(avg(amt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, amt as value from with_ma
union all select record_month, '3-Mo MA', ma_3 from with_ma
union all select record_month, '6-Mo MA', ma_6 from with_ma
order by 1, 2
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
union all select round(mean_val, 0), 'Mean' from stats
union all select greatest(0, round(mean_val - 3 * sigma, 0)), 'LCL' from stats
```

{% line_chart
    data="control_equity_dollars"
    x="record_month"
    y="value"
    series="series"
    series_order=["Monthly", "3-Mo MA", "6-Mo MA"]
    y_fmt="usd0"
    title="Equity Unlock Signals -- Dollar Volume Control Chart"
    subtitle="3-year window. Dashed lines = 3-sigma XmR limits."
    chart_options={
        series_colors = {
            "Monthly" = "#FF800E"
            "3-Mo MA" = "#FFBC79"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_equity_dollars_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}

**So what?** Is an above-normal rate of equity unlocks indicating borrower departures--could this be a retention issue worth investigating? If unlocks are rising alongside falling interest rates, could a refinance wave be driving them? Are the freed-collateral borrowers potential HELOC or win-back candidates? How tight is the outreach window before these borrowers commit to another lender?


### Owner Pressure Signals

**What's in this subcategory:** Liens, judgments, lis pendens, tax lien notices, mechanics liens, and writs of execution. Financial stress short of foreclosure. Weights: signal=9, distress=9, revenue=3.

**How to read:** High distress weight, low revenue weight. A judgment or tax lien indicates the owner is under financial pressure that may lead to sale, refinance, settlement, or escalation to foreclosure.

```sql control_pressure_volume
-- Full 3-year window.
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where subcategory = 'Owner Pressure Signals'
    group by 1
),
with_ma as (
    select record_month, cnt,
        round(avg(cnt) over (order by record_month rows between 2 preceding and current row), 0) as ma_3,
        round(avg(cnt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, cnt as value from with_ma
union all select record_month, '3-Mo MA', ma_3 from with_ma
union all select record_month, '6-Mo MA', ma_6 from with_ma
order by 1, 2
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
union all select round(mean_val, 0), 'Mean' from stats
union all select greatest(0, round(mean_val - 3 * sigma, 0)), 'LCL' from stats
```

{% line_chart
    data="control_pressure_volume"
    x="record_month"
    y="value"
    series="series"
    series_order=["Monthly", "3-Mo MA", "6-Mo MA"]
    y_fmt="num0"
    title="Owner Pressure Signals -- Volume Control Chart"
    subtitle="3-year window. Dashed lines = 3-sigma XmR limits. Highest distress weight subcategory."
    chart_options={
        series_colors = {
            "Monthly" = "#006BA4"
            "3-Mo MA" = "#5F9ED1"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_pressure_volume_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}

```sql control_pressure_dollars
-- Full 3-year window.
with monthly as (
    select record_month, sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where subcategory = 'Owner Pressure Signals'
    group by 1
),
with_ma as (
    select record_month, amt,
        round(avg(amt) over (order by record_month rows between 2 preceding and current row), 0) as ma_3,
        round(avg(amt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, amt as value from with_ma
union all select record_month, '3-Mo MA', ma_3 from with_ma
union all select record_month, '6-Mo MA', ma_6 from with_ma
order by 1, 2
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
union all select round(mean_val, 0), 'Mean' from stats
union all select greatest(0, round(mean_val - 3 * sigma, 0)), 'LCL' from stats
```

{% line_chart
    data="control_pressure_dollars"
    x="record_month"
    y="value"
    series="series"
    series_order=["Monthly", "3-Mo MA", "6-Mo MA"]
    y_fmt="usd0"
    title="Owner Pressure Signals -- Dollar Volume Control Chart"
    subtitle="3-year window. Dashed lines = 3-sigma XmR limits."
    chart_options={
        series_colors = {
            "Monthly" = "#FF800E"
            "3-Mo MA" = "#FFBC79"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_pressure_dollars_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}

**So what?** Could a sustained breakout above the UCL indicate a genuine increase in market-wide financial distress, not just a noisy month? If pressure is rising, what does the Blocker Clearance chart show--are these pressures being resolved or accumulating? Are there CRA or compliance implications if distress is concentrating in specific geographies? Is this trend visible in the Divergence Monitor above alongside equity unlocks?


### Forced Sale Signals

**What's in this subcategory:** Notices of default, notices of foreclosure, foreclosure deeds, marshal's deeds, tax deeds, trustee deeds, and power of sale deeds. Weights: signal=10, distress=10, revenue=4.

**How to read:** Maximum signal and distress weights. This subcategory contains both leading indicators (notices, which typically precede a forced sale) and lagging indicators (deeds, which mean the forced sale already occurred).

```sql control_forced_volume
-- Full 3-year window.
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where subcategory = 'Forced Sale Signals'
    group by 1
),
with_ma as (
    select record_month, cnt,
        round(avg(cnt) over (order by record_month rows between 2 preceding and current row), 0) as ma_3,
        round(avg(cnt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, cnt as value from with_ma
union all select record_month, '3-Mo MA', ma_3 from with_ma
union all select record_month, '6-Mo MA', ma_6 from with_ma
order by 1, 2
```

```sql control_forced_volume_limits
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where subcategory = 'Forced Sale Signals'
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
union all select round(mean_val, 0), 'Mean' from stats
union all select greatest(0, round(mean_val - 3 * sigma, 0)), 'LCL' from stats
```

{% line_chart
    data="control_forced_volume"
    x="record_month"
    y="value"
    series="series"
    series_order=["Monthly", "3-Mo MA", "6-Mo MA"]
    y_fmt="num0"
    title="Forced Sale Signals -- Volume Control Chart"
    subtitle="3-year window. Dashed lines = 3-sigma XmR limits. Highest distress subcategory."
    chart_options={
        series_colors = {
            "Monthly" = "#006BA4"
            "3-Mo MA" = "#5F9ED1"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_forced_volume_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}

```sql control_forced_dollars
-- Full 3-year window.
with monthly as (
    select record_month, sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where subcategory = 'Forced Sale Signals'
    group by 1
),
with_ma as (
    select record_month, amt,
        round(avg(amt) over (order by record_month rows between 2 preceding and current row), 0) as ma_3,
        round(avg(amt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, amt as value from with_ma
union all select record_month, '3-Mo MA', ma_3 from with_ma
union all select record_month, '6-Mo MA', ma_6 from with_ma
order by 1, 2
```

```sql control_forced_dollars_limits
with monthly as (
    select record_month, sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where subcategory = 'Forced Sale Signals'
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
union all select round(mean_val, 0), 'Mean' from stats
union all select greatest(0, round(mean_val - 3 * sigma, 0)), 'LCL' from stats
```

{% line_chart
    data="control_forced_dollars"
    x="record_month"
    y="value"
    series="series"
    series_order=["Monthly", "3-Mo MA", "6-Mo MA"]
    y_fmt="usd0"
    title="Forced Sale Signals -- Dollar Volume Control Chart"
    subtitle="3-year window. Dashed lines = 3-sigma XmR limits."
    chart_options={
        series_colors = {
            "Monthly" = "#FF800E"
            "3-Mo MA" = "#FFBC79"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_forced_dollars_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}

**So what?** Are notices (leading) rising while deeds (lagging) remain flat--could foreclosure filings be increasing without yet reaching completion? Could a rising trend here warrant pre-sale counseling outreach (CRA opportunity) or post-sale investor acquisition preparation? How does this compare to the Owner Pressure trend--is pressure converting to forced sales, or being resolved through other channels?


### Blocker Clearance Signals

**What's in this subcategory:** Cancellations, terminations, withdrawals, dismissals, and notices of cancellation. These are "signal resolvers"--they cancel prior negative events. Weights: signal=5, distress=3, revenue=5.

**How to read:** A cancellation following a lis pendens or a dismissal following a judgment means the prior blocker has been removed. Properties with recent blocker clearance may be newly transactable.

```sql control_blocker_volume
-- Full 3-year window.
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where subcategory = 'Blocker Clearance Signals'
    group by 1
),
with_ma as (
    select record_month, cnt,
        round(avg(cnt) over (order by record_month rows between 2 preceding and current row), 0) as ma_3,
        round(avg(cnt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, cnt as value from with_ma
union all select record_month, '3-Mo MA', ma_3 from with_ma
union all select record_month, '6-Mo MA', ma_6 from with_ma
order by 1, 2
```

```sql control_blocker_volume_limits
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where subcategory = 'Blocker Clearance Signals'
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
union all select round(mean_val, 0), 'Mean' from stats
union all select greatest(0, round(mean_val - 3 * sigma, 0)), 'LCL' from stats
```

{% line_chart
    data="control_blocker_volume"
    x="record_month"
    y="value"
    series="series"
    series_order=["Monthly", "3-Mo MA", "6-Mo MA"]
    y_fmt="num0"
    title="Blocker Clearance Signals -- Volume Control Chart"
    subtitle="3-year window. Dashed lines = 3-sigma XmR limits."
    chart_options={
        series_colors = {
            "Monthly" = "#006BA4"
            "3-Mo MA" = "#5F9ED1"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_blocker_volume_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}

**So what?** Is blocker clearance keeping pace with owner pressure? If pressure is rising but blocker clearance is flat, could blockers be accumulating across the market? Could a spike in clearances represent a batch of resolved litigation or a policy change accelerating lien releases? Are the cleared properties worth prioritizing for outreach?


### Life and Legal Triggers

**What's in this subcategory:** Death certificates, divorce decrees, probate distributions, powers of attorney, trusts, gift deeds, and partition deeds. Weights: signal=7, distress=6, revenue=7.

**How to read:** Life events that may force property decisions regardless of market conditions. These are external shocks with varying and uncertain timelines depending on estate complexity, family circumstances, and legal proceedings.

```sql control_life_volume
-- Full 3-year window.
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where subcategory = 'Life and Legal Triggers'
    group by 1
),
with_ma as (
    select record_month, cnt,
        round(avg(cnt) over (order by record_month rows between 2 preceding and current row), 0) as ma_3,
        round(avg(cnt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, cnt as value from with_ma
union all select record_month, '3-Mo MA', ma_3 from with_ma
union all select record_month, '6-Mo MA', ma_6 from with_ma
order by 1, 2
```

```sql control_life_volume_limits
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where subcategory = 'Life and Legal Triggers'
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
union all select round(mean_val, 0), 'Mean' from stats
union all select greatest(0, round(mean_val - 3 * sigma, 0)), 'LCL' from stats
```

{% line_chart
    data="control_life_volume"
    x="record_month"
    y="value"
    series="series"
    series_order=["Monthly", "3-Mo MA", "6-Mo MA"]
    y_fmt="num0"
    title="Life and Legal Triggers -- Volume Control Chart"
    subtitle="3-year window. Dashed lines = 3-sigma XmR limits."
    chart_options={
        series_colors = {
            "Monthly" = "#006BA4"
            "3-Mo MA" = "#5F9ED1"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_life_volume_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}

**So what?** Are life events trending up due to demographic shifts (aging population, estate settlements)? Could a spike in probate-related filings indicate a wave of estate properties that may enter the market? Are divorce-related signals creating property settlement opportunities? Since these events are market-independent, are they providing a stable baseline of transaction opportunities even during slow market periods?


### Deal Propensity Events

**What's in this subcategory:** Rescissions and amendments. Weights: signal=6, distress=1, revenue=6.

**How to read:** Low volume. An amendment indicates a deal is being restructured and is actively in progress. A rescission indicates a deal is unwinding.

```sql control_deal_prop_volume
-- Full 3-year window.
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where subcategory = 'Deal Propensity Events'
    group by 1
),
with_ma as (
    select record_month, cnt,
        round(avg(cnt) over (order by record_month rows between 2 preceding and current row), 0) as ma_3,
        round(avg(cnt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, cnt as value from with_ma
union all select record_month, '3-Mo MA', ma_3 from with_ma
union all select record_month, '6-Mo MA', ma_6 from with_ma
order by 1, 2
```

```sql control_deal_prop_volume_limits
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where subcategory = 'Deal Propensity Events'
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
union all select round(mean_val, 0), 'Mean' from stats
union all select greatest(0, round(mean_val - 3 * sigma, 0)), 'LCL' from stats
```

{% line_chart
    data="control_deal_prop_volume"
    x="record_month"
    y="value"
    series="series"
    series_order=["Monthly", "3-Mo MA", "6-Mo MA"]
    y_fmt="num0"
    title="Deal Propensity Events -- Volume Control Chart"
    subtitle="3-year window. Dashed lines = 3-sigma XmR limits."
    chart_options={
        series_colors = {
            "Monthly" = "#006BA4"
            "3-Mo MA" = "#5F9ED1"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_deal_prop_volume_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}

**So what?** Are rescissions rising--could deals be falling apart at higher rates? Are amendments rising--could that indicate more active deal restructuring? Given the low volume, is this subcategory more useful as a property-level indicator than a market-level trend?

---

## Signal Family: Transaction Feasibility & Close Intelligence

**The question this answers:** "Will this deal close, and what will block it?"

**What's in this family:** Three subcategories spanning title and ownership complexity (quitclaim deeds, affidavits, certificates, corrections), liens and encumbrances (lis pendens, tax rolls), and deal complexity events (assignments, addenda, stipulations). See Appendix A for full subcategory breakdown and weights.

### Category-Level Overview

```sql control_feasibility_volume
-- Full 3-year window. All 3 subcategories combined.
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where category = 'Transaction Feasibility & Close Intelligence'
    group by 1
),
with_ma as (
    select record_month, cnt,
        round(avg(cnt) over (order by record_month rows between 2 preceding and current row), 0) as ma_3,
        round(avg(cnt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, cnt as value from with_ma
union all select record_month, '3-Mo MA', ma_3 from with_ma
union all select record_month, '6-Mo MA', ma_6 from with_ma
order by 1, 2
```

```sql control_feasibility_volume_limits
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where category = 'Transaction Feasibility & Close Intelligence'
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
union all select round(mean_val, 0), 'Mean' from stats
union all select greatest(0, round(mean_val - 3 * sigma, 0)), 'LCL' from stats
```

{% line_chart
    data="control_feasibility_volume"
    x="record_month"
    y="value"
    series="series"
    series_order=["Monthly", "3-Mo MA", "6-Mo MA"]
    y_fmt="num0"
    title="Transaction Feasibility & Close -- Volume Control Chart"
    subtitle="All 3 subcategories combined. 3-year window."
    chart_options={
        series_colors = {
            "Monthly" = "#006BA4"
            "3-Mo MA" = "#5F9ED1"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_feasibility_volume_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}

```sql control_feasibility_dollars
-- Full 3-year window.
with monthly as (
    select record_month, sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where category = 'Transaction Feasibility & Close Intelligence'
    group by 1
),
with_ma as (
    select record_month, amt,
        round(avg(amt) over (order by record_month rows between 2 preceding and current row), 0) as ma_3,
        round(avg(amt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, amt as value from with_ma
union all select record_month, '3-Mo MA', ma_3 from with_ma
union all select record_month, '6-Mo MA', ma_6 from with_ma
order by 1, 2
```

```sql control_feasibility_dollars_limits
with monthly as (
    select record_month, sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where category = 'Transaction Feasibility & Close Intelligence'
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
union all select round(mean_val, 0), 'Mean' from stats
union all select greatest(0, round(mean_val - 3 * sigma, 0)), 'LCL' from stats
```

{% line_chart
    data="control_feasibility_dollars"
    x="record_month"
    y="value"
    series="series"
    series_order=["Monthly", "3-Mo MA", "6-Mo MA"]
    y_fmt="usd0"
    title="Transaction Feasibility & Close -- Dollar Volume Control Chart"
    subtitle="All 3 subcategories combined. 3-year window."
    chart_options={
        series_colors = {
            "Monthly" = "#FF800E"
            "3-Mo MA" = "#FFBC79"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_feasibility_dollars_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}

**So what?** Could a rising trend here indicate that deals across the market are getting harder to close? Are title companies, escrow, and lenders likely to see longer timelines or more fallouts? Which subcategory (title complexity, liens, or deal complexity) is driving the movement?


### Title and Ownership Complexity

**What's in this subcategory:** Quitclaim deeds, affidavits (identity, marital status, transferee, contract), certificates (title, estoppel), corrections, amended deeds, memoranda, and horizontal property regime documents. Weights: signal=2, distress=2, revenue=2.

**How to read:** These complicate ownership, title clarity, or property regime structure.

```sql control_title_volume
-- Full 3-year window.
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where subcategory = 'Title and Ownership Complexity'
    group by 1
),
with_ma as (
    select record_month, cnt,
        round(avg(cnt) over (order by record_month rows between 2 preceding and current row), 0) as ma_3,
        round(avg(cnt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, cnt as value from with_ma
union all select record_month, '3-Mo MA', ma_3 from with_ma
union all select record_month, '6-Mo MA', ma_6 from with_ma
order by 1, 2
```

```sql control_title_volume_limits
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where subcategory = 'Title and Ownership Complexity'
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
union all select round(mean_val, 0), 'Mean' from stats
union all select greatest(0, round(mean_val - 3 * sigma, 0)), 'LCL' from stats
```

{% line_chart
    data="control_title_volume"
    x="record_month"
    y="value"
    series="series"
    series_order=["Monthly", "3-Mo MA", "6-Mo MA"]
    y_fmt="num0"
    title="Title and Ownership Complexity -- Volume Control Chart"
    subtitle="3-year window. Dashed lines = 3-sigma XmR limits."
    chart_options={
        series_colors = {
            "Monthly" = "#006BA4"
            "3-Mo MA" = "#5F9ED1"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_title_volume_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}

**So what?** Could a rising trend indicate the deal pipeline is getting more complex--more quitclaim corrections, more affidavits needed to clear title? Is this correlated with the Life and Legal Triggers chart (estate settlements often generate title complexity)?


### Liens and Encumbrances

**What's in this subcategory:** Lis pendens, tax rolls, and lis pendens notices. Weights: signal=8, distress=8, revenue=3.

**How to read:** Active legal encumbrances that block or complicate transactions. A lis pendens signals ongoing litigation affecting the property. Tax rolls create senior liens that must be addressed before closing.

```sql control_liens_volume
-- Full 3-year window.
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where subcategory = 'Liens and Encumbrances'
    group by 1
),
with_ma as (
    select record_month, cnt,
        round(avg(cnt) over (order by record_month rows between 2 preceding and current row), 0) as ma_3,
        round(avg(cnt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, cnt as value from with_ma
union all select record_month, '3-Mo MA', ma_3 from with_ma
union all select record_month, '6-Mo MA', ma_6 from with_ma
order by 1, 2
```

```sql control_liens_volume_limits
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where subcategory = 'Liens and Encumbrances'
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
union all select round(mean_val, 0), 'Mean' from stats
union all select greatest(0, round(mean_val - 3 * sigma, 0)), 'LCL' from stats
```

{% line_chart
    data="control_liens_volume"
    x="record_month"
    y="value"
    series="series"
    series_order=["Monthly", "3-Mo MA", "6-Mo MA"]
    y_fmt="num0"
    title="Liens and Encumbrances -- Volume Control Chart"
    subtitle="3-year window. Dashed lines = 3-sigma XmR limits."
    chart_options={
        series_colors = {
            "Monthly" = "#006BA4"
            "3-Mo MA" = "#5F9ED1"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_liens_volume_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}

**So what?** Are new encumbrances being filed faster than they are being cleared (compare to Blocker Clearance)? Could a rising trend indicate increasing litigation or tax delinquency affecting the transactable property pool? What share of active listings or pipeline deals might be affected?


### Deal Complexity Events

**What's in this subcategory:** Assignments (lease, rents, amendment), addenda, and stipulations. Weights: signal=4, distress=1, revenue=4.

**How to read:** These restructure deal terms or introduce complexity. An assignment of rents signals an income property may be changing hands. A stipulation signals negotiated terms in litigation affecting property.

```sql control_deal_complex_volume
-- Full 3-year window.
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where subcategory = 'Deal Complexity Events'
    group by 1
),
with_ma as (
    select record_month, cnt,
        round(avg(cnt) over (order by record_month rows between 2 preceding and current row), 0) as ma_3,
        round(avg(cnt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, cnt as value from with_ma
union all select record_month, '3-Mo MA', ma_3 from with_ma
union all select record_month, '6-Mo MA', ma_6 from with_ma
order by 1, 2
```

```sql control_deal_complex_volume_limits
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where subcategory = 'Deal Complexity Events'
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
union all select round(mean_val, 0), 'Mean' from stats
union all select greatest(0, round(mean_val - 3 * sigma, 0)), 'LCL' from stats
```

{% line_chart
    data="control_deal_complex_volume"
    x="record_month"
    y="value"
    series="series"
    series_order=["Monthly", "3-Mo MA", "6-Mo MA"]
    y_fmt="num0"
    title="Deal Complexity Events -- Volume Control Chart"
    subtitle="3-year window. Dashed lines = 3-sigma XmR limits."
    chart_options={
        series_colors = {
            "Monthly" = "#006BA4"
            "3-Mo MA" = "#5F9ED1"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_deal_complex_volume_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}

**So what?** Are rent assignments increasing--could that signal rising investor activity in income properties? Are stipulations increasing--could that indicate more litigation-affected properties entering negotiated resolution?

---

## Signal Family: Market Dynamics Intelligence

**The question this answers:** "Where is the market tightening or loosening?"

**What's in this family:** Zone changes, land use commission actions, permits, easements, and declarations of taking. Supply pipeline signals. See Appendix A for weights.

**How to read:** These are early indicators of new inventory potentially entering the market. Low volume individually, but when aggregated they may reveal shifts in development activity.

```sql control_market_volume
-- Full 3-year window.
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where category = 'Market Dynamics Intelligence'
    group by 1
),
with_ma as (
    select record_month, cnt,
        round(avg(cnt) over (order by record_month rows between 2 preceding and current row), 0) as ma_3,
        round(avg(cnt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, cnt as value from with_ma
union all select record_month, '3-Mo MA', ma_3 from with_ma
union all select record_month, '6-Mo MA', ma_6 from with_ma
order by 1, 2
```

```sql control_market_volume_limits
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where category = 'Market Dynamics Intelligence'
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
union all select round(mean_val, 0), 'Mean' from stats
union all select greatest(0, round(mean_val - 3 * sigma, 0)), 'LCL' from stats
```

{% line_chart
    data="control_market_volume"
    x="record_month"
    y="value"
    series="series"
    series_order=["Monthly", "3-Mo MA", "6-Mo MA"]
    y_fmt="num0"
    title="Market Dynamics Intelligence -- Volume Control Chart"
    subtitle="3-year window. Dashed lines = 3-sigma XmR limits."
    chart_options={
        series_colors = {
            "Monthly" = "#006BA4"
            "3-Mo MA" = "#5F9ED1"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_market_volume_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}

**So what?** Could a zone change today lead to construction permits and new housing supply--and if so, on what timeline? What assumptions about Guam's permitting lifecycle, funding availability, and construction capacity would need to hold for that to materialize? Are easement filings concentrated in specific areas that could indicate targeted development corridors? Is this family useful primarily as a geographic signal (where) rather than a timing signal (when)?

---

## Signal Family: Regulatory, Risk & Claims Intelligence

**The question this answers:** "Where are compliance and loss risks concentrated?"

**What's in this family:** Waivers, notices of taking, designations, declarations, covenants, and assessments. Weights: signal=3, distress=3, revenue=1. Low direct revenue value but potentially important for portfolio risk monitoring and compliance. See Appendix A for the full instrument type list.

**How to read:** Low volume. Assessments create additional financial burden on property owners. Covenants and declarations impose usage restrictions and title complexity. A notice of taking signals eminent domain.

```sql control_regulatory_volume
-- Full 3-year window.
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where category = 'Regulatory, Risk & Claims Intelligence'
    group by 1
),
with_ma as (
    select record_month, cnt,
        round(avg(cnt) over (order by record_month rows between 2 preceding and current row), 0) as ma_3,
        round(avg(cnt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, cnt as value from with_ma
union all select record_month, '3-Mo MA', ma_3 from with_ma
union all select record_month, '6-Mo MA', ma_6 from with_ma
order by 1, 2
```

```sql control_regulatory_volume_limits
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where category = 'Regulatory, Risk & Claims Intelligence'
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
union all select round(mean_val, 0), 'Mean' from stats
union all select greatest(0, round(mean_val - 3 * sigma, 0)), 'LCL' from stats
```

{% line_chart
    data="control_regulatory_volume"
    x="record_month"
    y="value"
    series="series"
    series_order=["Monthly", "3-Mo MA", "6-Mo MA"]
    y_fmt="num0"
    title="Regulatory, Risk & Claims -- Volume Control Chart"
    subtitle="3-year window. Dashed lines = 3-sigma XmR limits."
    chart_options={
        series_colors = {
            "Monthly" = "#006BA4"
            "3-Mo MA" = "#5F9ED1"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_regulatory_volume_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}

**So what?** Could rising assessments increase delinquency risk for overlapping properties in the portfolio? Are notices of taking concentrated in areas with active infrastructure projects? Is this category worth monitoring at the individual-instrument level rather than the aggregate--given the low volume, each filing may warrant individual review?

---

## Annual Trend

**How to read:** Year-over-year volume by category. The pivot table shows instrument counts per year per category. The bar chart shows Capital & Leverage dollar volume per year. The second table breaks Transaction Propensity into subcategories.

**Time window:** Full history as defined by the `gold.yearly_instrument_summary` view. Metadata excluded from the pivot table.

{% table
    data="gold_yearly_instrument_summary"
    where="category != 'Metadata'"
    page_size=50
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

{% table
    data="gold_yearly_instrument_summary"
    where="category = 'Transaction Propensity Intelligence'"
    page_size=50
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

**So what?** Which years saw peak lending activity--and do those peaks correlate with known interest rate environments or policy changes? Are any categories showing multi-year structural shifts rather than cyclical variation? Is the current year's pace (see YTD table above) consistent with the multi-year trajectory, or is it diverging?

---

## Appendix A: Signal Classification Reference

RealFACTS classifies every recorded instrument into one of five signal families plus a Metadata category, using the `dim.signal_category` dimension table. Classification is driven by the instrument's type and sub-type fields joined to the dimension table in the bronze layer. Current coverage metrics are available live in the Platform Snapshot at the top of this page and in the audit layer (`audit.signal_category_coverage`).

### Categories

| Category | What It Contains | Typical Weights (S/D/R) |
|---|---|---|
| **Capital & Leverage Intelligence** | Mortgages, modifications, subordinations, bonds, promissory notes, assumptions | 10 / 2 / 10 |
| **Transaction Propensity Intelligence** | Deeds, leases, contracts, sales, releases, liens, judgments, probate, forced sales | Varies by subcategory |
| **Transaction Feasibility & Close Intelligence** | Quitclaim deeds, affidavits, title certificates, assignments, lis pendens, corrections | 2-8 / 1-8 / 2-4 |
| **Market Dynamics Intelligence** | Zone changes, permits, easements, declarations of taking, bills of sale | 3-8 / 1 / 3-8 |
| **Regulatory, Risk & Claims Intelligence** | Waivers, notices of taking, designations, declarations, covenants, assessments | 3 / 3 / 1 |
| **Metadata** | Administrative filings, bylaws, articles, minutes, certificates of title | 1 / 1 / 1 |

### Transaction Propensity Subcategories

| Subcategory | What It Contains | Weights (S/D/R) |
|---|---|---|
| **Transaction Timing Triggers** | Deeds, conveyances, leases, contracts, options, maps | 8 / 1 / 8 |
| **Equity Unlock Signals** | Mortgage releases, satisfactions, discharges, judgment cancellations | 9 / 1 / 9 |
| **Forced Sale Signals** | Notices of default/foreclosure, tax deeds, trustee deeds, power of sale | 10 / 10 / 4 |
| **Owner Pressure Signals** | Liens, judgments, lis pendens, tax lien notices, mechanics liens, writs | 9 / 9 / 3 |
| **Blocker Clearance Signals** | Cancellations, terminations, withdrawals, dismissals | 5 / 3 / 5 |
| **Life and Legal Triggers** | Death certificates, divorce decrees, probate, powers of attorney, trusts, gift deeds | 7 / 6 / 7 |
| **Deal Propensity Events** | Rescissions, amendments | 6 / 1 / 6 |

### Transaction Feasibility Subcategories

| Subcategory | What It Contains | Weights (S/D/R) |
|---|---|---|
| **Title and Ownership Complexity** | Quitclaim deeds, affidavits, certificates, corrections, memoranda | 2 / 2 / 2 |
| **Liens and Encumbrances** | Lis pendens, tax rolls | 8 / 8 / 3 |
| **Deal Complexity Events** | Assignments (lease, rents), addenda, stipulations | 4 / 1 / 4 |

### Signal Weights

Each instrument type carries three weights on a 1-10 scale, as defined in `dim.signal_category`:

- **Signal weight (S):** General indicator of property activity.
- **Distress weight (D):** Strength of financial or legal pressure indication.
- **Revenue weight (R):** Directness of connection to a lending opportunity.

These three axes feed the weighted scoring engine in the silver and gold layers, enabling work queues to be tuned for different objectives: a distress counseling queue weights D heavily; a refinance capture queue weights R heavily.

---

## Appendix B: Control Chart Method

Every control chart on this page uses XmR (Individuals and Moving Range) analysis.

XmR answers one question: **is this month's number actually different, or is it just normal variation?**

It works by measuring how much the count typically jumps from one month to the next. That jump is the Moving Range--the absolute difference between consecutive months. From the average jump size, the method calculates how wide the normal variation band is. The average moving range is divided by 1.128 (the d2 constant for n=2) to estimate one standard deviation. Control limits are drawn at three standard deviations above and below the mean:

- **Center line (Mean):** Average of all monthly values
- **UCL (Upper Control Limit):** Mean + 3 standard deviations
- **LCL (Lower Control Limit):** Mean - 3 standard deviations (floored at zero)

Anything inside the band is expected variation. Anything outside has less than a 0.27% probability under stable conditions.

**Time window:** All control charts use a 3-year window, defined by the underlying `gold.signal_velocity_trends` view (which reads from `silver.monthly_signal_velocity`, filtered to `CURRENT_DATE - INTERVAL '3' YEAR`).

### Moving Averages

Two smoothing lines overlay each chart:

- **3-month moving average** captures short-term shifts.
- **6-month moving average** reveals medium-term structural trend.

When the monthly value diverges from both moving averages, that divergence is itself a signal--even if the monthly value stays within control limits.

### Color Convention

- **Volume charts:** Navy (#006BA4) monthly, medium blue (#5F9ED1) 3-Mo MA, gray (#898989) 6-Mo MA
- **Dollar charts:** Orange (#FF800E) monthly, light amber (#FFBC79) 3-Mo MA, gray (#898989) 6-Mo MA
- **Control limits:** Dark gray dashed (#595959) for UCL, Mean, LCL

This palette maintains clear contrast for readers with color vision deficiencies.