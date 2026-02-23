---
name: Home
assetId: c18a30f5-1d9f-4e71-91d9-e5161bcc6957
type: page
---

# Executive Signals Overview

Every property on Guam tells a story through its recorded instruments. A mortgage filed today is a lending relationship born. A lien recorded tomorrow is financial pressure building. A satisfaction of mortgage next week is a borrower who just freed their collateral -- and is now either a competitor's customer or your next HELOC opportunity.

RealFACTS reads these stories at scale. The platform processes 993,000 recorded instruments from the Guam Department of Land Management, classifying each one into signal families that answer five questions a lending partner cares about: Who will transact next? Will this deal close? What's the capital position? Where is the market moving? Where are the compliance risks?

This page is the executive nerve center. It starts with what needs your attention right now, then unfolds category by category -- showing you not just what happened, but whether the change is real or just noise.

---

## Attention Now: Signals Outside Normal Range

Not every month-over-month change matters. These control charts scan every signal subcategory and flag only the ones where the most recent complete month fell outside statistical control limits -- a less than 0.3% probability event under stable conditions. If nothing appears here, the market is behaving normally. If something does, it warrants investigation.

```sql out_of_control_summary
with monthly as (
    select
        category,
        subcategory,
        record_month,
        sum(current_month_count) as cnt,
        sum(current_month_amount) as amt
    from gold_signal_velocity_trends
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
latest as (
    select
        m.category,
        m.subcategory,
        m.record_month,
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
    where m.record_month = (
        select distinct record_month
        from gold_signal_velocity_trends
        order by record_month desc
        limit 1 offset 1
    )
)
select
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

{% table data="out_of_control_summary" /%}

---

## Latest Month at a Glance

```sql latest_month_kpis
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
        category, subcategory,
        sum(current_month_count) as cnt,
        sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where record_month = (select mo from current_month)
    group by 1, 2
),
prev as (
    select
        category, subcategory,
        sum(current_month_count) as cnt,
        sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where record_month = (select mo from prior_month)
    group by 1, 2
)
select
    'Total Signals' as metric,
    (select sum(cnt)::double from curr where category != 'Metadata') as current_month,
    (select sum(cnt)::double from prev where category != 'Metadata') as prior_month,
    (select sum(cnt)::double from curr where category != 'Metadata')
    - (select sum(cnt)::double from prev where category != 'Metadata') as delta

union all

select 'Owner Pressure',
    (select sum(cnt)::double from curr where subcategory = 'Owner Pressure Signals'),
    (select sum(cnt)::double from prev where subcategory = 'Owner Pressure Signals'),
    (select sum(cnt)::double from curr where subcategory = 'Owner Pressure Signals')
    - (select sum(cnt)::double from prev where subcategory = 'Owner Pressure Signals')

union all

select 'Equity Unlocks',
    (select sum(cnt)::double from curr where subcategory = 'Equity Unlock Signals'),
    (select sum(cnt)::double from prev where subcategory = 'Equity Unlock Signals'),
    (select sum(cnt)::double from curr where subcategory = 'Equity Unlock Signals')
    - (select sum(cnt)::double from prev where subcategory = 'Equity Unlock Signals')

union all

select 'Capital & Leverage Volume ($)',
    (select sum(amt)::double from curr where category = 'Capital & Leverage Intelligence'),
    (select sum(amt)::double from prev where category = 'Capital & Leverage Intelligence'),
    (select sum(amt)::double from curr where category = 'Capital & Leverage Intelligence')
    - (select sum(amt)::double from prev where category = 'Capital & Leverage Intelligence')
```

{% row %}
    {% big_value
        data="latest_month_kpis"
        value="total_signals"
        title="Total Signals (Latest Month)"
        fmt="num0"
        comparison="prior_total_signals"
        comparison_title="vs prior month"
    /%}
    {% big_value
        data="latest_month_kpis"
        value="transaction_propensity"
        title="Transaction Propensity"
        fmt="num0"
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
    /%}
    {% big_value
        data="latest_month_kpis"
        value="blockers_cleared"
        title="Blockers Cleared"
        fmt="num0"
    /%}
{% /row %}

### Month-over-Month Comparison

```sql period_comparison
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
        category, subcategory,
        sum(current_month_count) as cnt,
        sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where record_month = (select mo from current_month)
    group by 1, 2
),
prev as (
    select
        category, subcategory,
        sum(current_month_count) as cnt,
        sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where record_month = (select mo from prior_month)
    group by 1, 2
)
select
    'Total Signals' as metric,
    (select sum(cnt) from curr where category != 'Metadata') as current_month,
    (select sum(cnt) from prev where category != 'Metadata') as prior_month,
    (select sum(cnt) from curr where category != 'Metadata')
    - (select sum(cnt) from prev where category != 'Metadata') as delta

union all

select 'Owner Pressure',
    (select sum(cnt) from curr where subcategory = 'Owner Pressure Signals'),
    (select sum(cnt) from prev where subcategory = 'Owner Pressure Signals'),
    (select sum(cnt) from curr where subcategory = 'Owner Pressure Signals')
    - (select sum(cnt) from prev where subcategory = 'Owner Pressure Signals')

union all

select 'Equity Unlocks',
    (select sum(cnt) from curr where subcategory = 'Equity Unlock Signals'),
    (select sum(cnt) from prev where subcategory = 'Equity Unlock Signals'),
    (select sum(cnt) from curr where subcategory = 'Equity Unlock Signals')
    - (select sum(cnt) from prev where subcategory = 'Equity Unlock Signals')

union all

select 'Capital & Leverage Volume ($)',
    (select sum(amt) from curr where category = 'Capital & Leverage Intelligence'),
    (select sum(amt) from prev where category = 'Capital & Leverage Intelligence'),
    (select sum(amt) from curr where category = 'Capital & Leverage Intelligence')
    - (select sum(amt) from prev where category = 'Capital & Leverage Intelligence')
```

{% table data="period_comparison" /%}

---

## Divergence Monitor: Paired Signals That Tell a Story

Individual signals are informative. Pairs of signals moving in opposite directions are diagnostic. These charts overlay two related subcategories so you can see divergence at a glance. Lighter lines show 6-month moving averages to reveal the underlying trend beneath monthly noise.

### Equity Unlocks vs Owner Pressure

When equity unlocks rise and owner pressure falls, borrowers are paying off voluntarily -- healthy market. When both rise together, borrowers are being forced to liquidate. When pressure rises and unlocks fall, stress is building with no relief valve.

```sql divergence_equity_pressure
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
        round(avg(cnt) over (partition by subcategory order by record_month rows between 5 preceding and current row), 0) as ma_6
    from base
)
select record_month, subcategory as series, cnt as monthly_count from with_ma
union all select record_month, subcategory || ' (6-Mo MA)', ma_6 from with_ma
order by 1, 2
```

{% line_chart
    data="divergence_equity_pressure"
    x="record_month"
    y="monthly_count"
    series="series"
    series_order=["Equity Unlock Signals", "Owner Pressure Signals", "Equity Unlock Signals (6-Mo MA)", "Owner Pressure Signals (6-Mo MA)"]
    y_fmt="num0"
    title="Equity Unlocks vs Owner Pressure -- Monthly Volume"
    subtitle="Divergence = diagnostic. Same direction = reinforcing. Lighter lines = 6-month smoothed trend."
    chart_options={
        series_colors = {
            "Equity Unlock Signals" = "#2CA02C"
            "Equity Unlock Signals (6-Mo MA)" = "#B5E0B5"
            "Owner Pressure Signals" = "#D62728"
            "Owner Pressure Signals (6-Mo MA)" = "#F2B4B4"
        }
    }
/%}

### Capital & Leverage vs Forced Sales

New mortgage originations alongside low forced sales means a growing, healthy market. Falling originations with rising forced sales means the credit cycle is turning. This pair is the clearest leading indicator of market health.

```sql divergence_capital_forced
with base as (
    select
        record_month,
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
        round(avg(cnt) over (partition by signal_name order by record_month rows between 5 preceding and current row), 0) as ma_6
    from base
)
select record_month, signal_name as series, cnt as monthly_count from with_ma
union all select record_month, signal_name || ' (6-Mo MA)', ma_6 from with_ma
order by 1, 2
```

{% line_chart
    data="divergence_capital_forced"
    x="record_month"
    y="monthly_count"
    series="series"
    series_order=["Capital & Leverage", "Forced Sale Signals", "Capital & Leverage (6-Mo MA)", "Forced Sale Signals (6-Mo MA)"]
    y_fmt="num0"
    title="Capital & Leverage vs Forced Sales -- Monthly Volume"
    subtitle="Healthy market = high originations, low forced sales. Turning market = convergence."
    chart_options={
        series_colors = {
            "Capital & Leverage" = "#006BA4"
            "Capital & Leverage (6-Mo MA)" = "#A2C8EC"
            "Forced Sale Signals" = "#D62728"
            "Forced Sale Signals (6-Mo MA)" = "#F2B4B4"
        }
    }
/%}

---

## Market Pulse: All Signal Categories

The broadest view. Every recorded instrument classified into one of five signal families (Metadata excluded), tracked monthly.

```sql market_pulse_summary
with monthly as (
    select
        record_month,
        category,
        sum(current_month_count) as cnt,
        sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where category != 'Metadata'
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

{% table data="market_pulse_summary" /%}

---

## YTD Pace vs Prior Year

Is this year on track? This chart compares the current year's annualized pace (year-to-date count divided by months elapsed, multiplied by 12) against last year's full-year total for each category.

```sql ytd_pace
with current_yr as (
    select
        category,
        sum(instrument_count) as ytd_count
    from gold_yearly_instrument_summary
    where recorded_year = (select max(recorded_year) from gold_yearly_instrument_summary)
      and category != 'Metadata'
    group by 1
),
prior_yr as (
    select
        category,
        sum(instrument_count) as full_year_count
    from gold_yearly_instrument_summary
    where recorded_year = (select max(recorded_year) - 1 from gold_yearly_instrument_summary)
      and category != 'Metadata'
    group by 1
),
months_elapsed as (
    select count(distinct record_month) as months_so_far
    from gold_signal_velocity_trends
    where record_month >= date_trunc('year', (select max(record_month) from gold_signal_velocity_trends))
)
select
    c.category,
    'Annualized YTD Pace' as measure,
    round(c.ytd_count * 12.0 / nullif(m.months_so_far, 0), 0) as instrument_count
from current_yr c
cross join months_elapsed m

union all

select
    p.category,
    'Prior Year Total',
    p.full_year_count
from prior_yr p

order by 2, instrument_count desc
```

{% bar_chart
    data="ytd_pace"
    x="category"
    y="instrument_count"
    series="measure"
    y_fmt="num0"
    title="Annualized YTD Pace vs Prior Year Total"
    subtitle="Side-by-side comparison. Is each category accelerating or decelerating year over year?"
    chart_options={
        series_colors = {
            "Annualized YTD Pace" = "#006BA4"
            "Prior Year Total" = "#FFBC79"
        }
    }
/%}

{% table data="ytd_pace" /%}

---

## Signal Concentration: Where Does the Volume Live?

A handful of subcategories drive the majority of both volume and dollar exposure. These Pareto tables identify the top 80% -- the ones worth watching most closely. Everything that follows drills into these subcategories one by one.

### Volume Pareto -- Last 12 Months

```sql pareto_volume
with subcategory_totals as (
    select
        subcategory,
        sum(current_month_count) as total_count
    from gold_signal_velocity_trends
    where category != 'Metadata'
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

{% table data="pareto_volume" /%}

### Dollar Pareto -- Last 12 Months

```sql pareto_dollars
with subcategory_totals as (
    select
        subcategory,
        sum(current_month_amount) as total_amount
    from gold_signal_velocity_trends
    where category != 'Metadata'
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

{% table data="pareto_dollars" /%}

---

## Signal Family: Capital & Leverage Intelligence

**The question this answers:** "What's the capital position, and what financial actions are optimal now?"

Every mortgage, modification, subordination, bond, promissory note, and assumption recorded on Guam flows through this signal family. It is the direct heartbeat of lending activity -- when this number rises, more money is being deployed into Guam real estate. When it falls, the market is contracting or borrowers are going elsewhere.

Capital & Leverage carries the highest weights in the system: signal=10, distress=2, revenue=10. It is low-distress by nature (a new mortgage is not a distress event) but maximum signal and maximum revenue. This is the lending partner's core book of business.

The 17 instrument types in this family include new mortgage originations, modifications (which may signal either distress-driven forbearance or opportunity-driven rate adjustment), subordination agreements (layered debt, complex capital structures), assumptions (ownership change without new financing, common in divorce and estate settlement), and assignment variants tracking servicing transfers.

**What to watch for:** A sustained drop below the lower control limit means lending activity has genuinely contracted -- not a seasonal dip but a structural shift. A spike above the upper limit could signal a refi wave (check against interest rate environment) or a surge in construction lending (check against Market Dynamics signals).

```sql control_capital_leverage_volume
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where category = 'Capital & Leverage Intelligence'
    group by 1
),
with_ma as (
    select record_month, cnt,
        round(avg(cnt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, cnt as value from with_ma
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
    series_order=["Monthly", "6-Mo MA"]
    y_fmt="num0"
    title="Capital & Leverage Intelligence -- Volume Control Chart"
    subtitle="Points outside UCL/LCL represent statistically significant shifts in lending activity"
    chart_options={
        series_colors = {
            "Monthly" = "#006BA4"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_capital_leverage_volume_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}

```sql control_capital_leverage_dollars
with monthly as (
    select record_month, sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where category = 'Capital & Leverage Intelligence'
    group by 1
),
with_ma as (
    select record_month, amt,
        round(avg(amt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, amt as value from with_ma
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
    series_order=["Monthly", "6-Mo MA"]
    y_fmt="usd0"
    title="Capital & Leverage Intelligence -- Dollar Volume Control Chart"
    chart_options={
        series_colors = {
            "Monthly" = "#D62728"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_capital_leverage_dollars_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}

---

## Signal Family: Transaction Propensity Intelligence

**The question this answers:** "Who will transact next, and why?"

This is the largest and most complex signal family -- 93 instrument types across 7 subcategories. Realtors, brokers, and lenders waste time on low-probability prospects while missing high-probability opportunities because they lack timing intelligence. Transaction Propensity predicts which properties are likely to sell, refinance, or experience forced sale, and explains the driver.

The seven subcategories form a narrative arc from healthy market activity (Transaction Timing Triggers) through escalating distress (Owner Pressure, Forced Sales) to resolution (Blocker Clearance, Equity Unlock). Life and Legal Triggers sit outside this arc as external shocks -- death, divorce, probate -- that force property decisions regardless of market conditions.

### Category-Level Overview

```sql control_txn_propensity_volume
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where category = 'Transaction Propensity Intelligence'
    group by 1
),
with_ma as (
    select record_month, cnt,
        round(avg(cnt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, cnt as value from with_ma
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
    series_order=["Monthly", "6-Mo MA"]
    y_fmt="num0"
    title="Transaction Propensity Intelligence -- Volume Control Chart"
    subtitle="All 7 subcategories combined"
    chart_options={
        series_colors = {
            "Monthly" = "#006BA4"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_txn_propensity_volume_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}

```sql control_txn_propensity_dollars
with monthly as (
    select record_month, sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where category = 'Transaction Propensity Intelligence'
    group by 1
),
with_ma as (
    select record_month, amt,
        round(avg(amt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, amt as value from with_ma
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
    series_order=["Monthly", "6-Mo MA"]
    y_fmt="usd0"
    title="Transaction Propensity Intelligence -- Dollar Volume Control Chart"
    chart_options={
        series_colors = {
            "Monthly" = "#D62728"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_txn_propensity_dollars_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}


### Transaction Timing Triggers

Deeds, conveyances, leases, contracts, options, maps, and bills of sale. These are the active market transactions -- the deposits pipeline, the sales outreach targets, the new relationship opportunities. A recent deed transfer or notice of sale indicates high probability the property will transact again within 24-36 months (ownership lifecycle patterns). Option exercise signals imminent transaction within 30-90 days. Weights: signal=8, distress=1, revenue=8.

```sql control_timing_volume
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where subcategory = 'Transaction Timing Triggers'
    group by 1
),
with_ma as (
    select record_month, cnt,
        round(avg(cnt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, cnt as value from with_ma
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
    series_order=["Monthly", "6-Mo MA"]
    y_fmt="num0"
    title="Transaction Timing Triggers -- Volume Control Chart"
    chart_options={
        series_colors = {
            "Monthly" = "#006BA4"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_timing_volume_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}

```sql control_timing_dollars
with monthly as (
    select record_month, sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where subcategory = 'Transaction Timing Triggers'
    group by 1
),
with_ma as (
    select record_month, amt,
        round(avg(amt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, amt as value from with_ma
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
    series_order=["Monthly", "6-Mo MA"]
    y_fmt="usd0"
    title="Transaction Timing Triggers -- Dollar Volume Control Chart"
    chart_options={
        series_colors = {
            "Monthly" = "#D62728"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_timing_dollars_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}


### Equity Unlock Signals

Mortgage releases, satisfactions, discharges, and judgment cancellations. Every equity unlock is a borrower who just freed their collateral. They either refinanced with a competitor (retention failure -- win-back opportunity), sold the property (too late for this one), or paid off entirely (HELOC opportunity). The conversion window is tight: 30-90 days for highest probability. Weights: signal=9, distress=1, revenue=9.

```sql control_equity_volume
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where subcategory = 'Equity Unlock Signals'
    group by 1
),
with_ma as (
    select record_month, cnt,
        round(avg(cnt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, cnt as value from with_ma
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
    series_order=["Monthly", "6-Mo MA"]
    y_fmt="num0"
    title="Equity Unlock Signals -- Volume Control Chart"
    subtitle="Breakout above UCL = abnormal rate of borrower departures"
    chart_options={
        series_colors = {
            "Monthly" = "#006BA4"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_equity_volume_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}

```sql control_equity_dollars
with monthly as (
    select record_month, sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where subcategory = 'Equity Unlock Signals'
    group by 1
),
with_ma as (
    select record_month, amt,
        round(avg(amt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, amt as value from with_ma
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
    series_order=["Monthly", "6-Mo MA"]
    y_fmt="usd0"
    title="Equity Unlock Signals -- Dollar Volume Control Chart"
    chart_options={
        series_colors = {
            "Monthly" = "#D62728"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_equity_dollars_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}


### Owner Pressure Signals

Liens, judgments, lis pendens, tax lien notices, mechanics liens, and writs of execution. Financial stress short of foreclosure. A judgment filed means the owner is under financial pressure and may need to sell, refinance to pay it off, or settle. A tax lien means government pressure requiring resolution or facing foreclosure. Timing: immediate pressure at 30-90 days, resolution pathway at 6-12 months. Weights: signal=9, distress=9, revenue=3.

This is the early warning system for CRA exposure. A sustained breakout above the UCL means distress is genuinely increasing across the market, not just a noisy month.

```sql control_pressure_volume
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where subcategory = 'Owner Pressure Signals'
    group by 1
),
with_ma as (
    select record_month, cnt,
        round(avg(cnt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, cnt as value from with_ma
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
    series_order=["Monthly", "6-Mo MA"]
    y_fmt="num0"
    title="Owner Pressure Signals -- Volume Control Chart"
    subtitle="Breakout above UCL = statistically significant increase in financial distress"
    chart_options={
        series_colors = {
            "Monthly" = "#006BA4"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_pressure_volume_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}

```sql control_pressure_dollars
with monthly as (
    select record_month, sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where subcategory = 'Owner Pressure Signals'
    group by 1
),
with_ma as (
    select record_month, amt,
        round(avg(amt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, amt as value from with_ma
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
    series_order=["Monthly", "6-Mo MA"]
    y_fmt="usd0"
    title="Owner Pressure Signals -- Dollar Volume Control Chart"
    chart_options={
        series_colors = {
            "Monthly" = "#D62728"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_pressure_dollars_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}


### Forced Sale Signals

Notices of default, notices of foreclosure, foreclosure deeds, marshal's deeds, tax deeds, trustee deeds, and power of sale deeds. This is the highest-distress subcategory in the system. A notice of default initiates foreclosure proceedings with high probability of forced sale in 90-180 days. A foreclosure deed means the transaction already occurred -- the opportunity is either pre-sale counseling (CRA) or post-sale investor acquisition. Weights: signal=10, distress=10, revenue=4.

```sql control_forced_volume
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where subcategory = 'Forced Sale Signals'
    group by 1
),
with_ma as (
    select record_month, cnt,
        round(avg(cnt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, cnt as value from with_ma
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
    series_order=["Monthly", "6-Mo MA"]
    y_fmt="num0"
    title="Forced Sale Signals -- Volume Control Chart"
    subtitle="Highest distress subcategory. Breakout = genuine shift in foreclosure activity."
    chart_options={
        series_colors = {
            "Monthly" = "#006BA4"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_forced_volume_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}

```sql control_forced_dollars
with monthly as (
    select record_month, sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where subcategory = 'Forced Sale Signals'
    group by 1
),
with_ma as (
    select record_month, amt,
        round(avg(amt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, amt as value from with_ma
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
    series_order=["Monthly", "6-Mo MA"]
    y_fmt="usd0"
    title="Forced Sale Signals -- Dollar Volume Control Chart"
    chart_options={
        series_colors = {
            "Monthly" = "#D62728"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_forced_dollars_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}


### Blocker Clearance Signals

Cancellations, terminations, withdrawals, dismissals, and notices of cancellation. These are "signal resolvers" -- they cancel prior negative events. A cancellation following a lis pendens means the litigation ended. A dismissal following a judgment means the pressure is off. Properties with recent blocker clearance are newly transactable -- fresh opportunities for outreach. Weights: signal=5, distress=3, revenue=5.

```sql control_blocker_volume
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where subcategory = 'Blocker Clearance Signals'
    group by 1
),
with_ma as (
    select record_month, cnt,
        round(avg(cnt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, cnt as value from with_ma
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
    series_order=["Monthly", "6-Mo MA"]
    y_fmt="num0"
    title="Blocker Clearance Signals -- Volume Control Chart"
    chart_options={
        series_colors = {
            "Monthly" = "#006BA4"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_blocker_volume_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}

```sql control_blocker_dollars
with monthly as (
    select record_month, sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where subcategory = 'Blocker Clearance Signals'
    group by 1
),
with_ma as (
    select record_month, amt,
        round(avg(amt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, amt as value from with_ma
union all select record_month, '6-Mo MA', ma_6 from with_ma
order by 1, 2
```

```sql control_blocker_dollars_limits
with monthly as (
    select record_month, sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where subcategory = 'Blocker Clearance Signals'
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
    data="control_blocker_dollars"
    x="record_month"
    y="value"
    series="series"
    series_order=["Monthly", "6-Mo MA"]
    y_fmt="usd0"
    title="Blocker Clearance Signals -- Dollar Volume Control Chart"
    chart_options={
        series_colors = {
            "Monthly" = "#D62728"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_blocker_dollars_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}


### Life and Legal Triggers

Death certificates, divorce decrees, probate distributions, powers of attorney, trusts, gift deeds, and partition deeds. Life events that force property decisions regardless of market conditions. Death and probate mean an estate must be settled -- property is often sold to distribute assets (6-18 month horizon). Divorce means property settlement -- one spouse buys out the other or the property is sold (6-12 months post-decree). Trust formation may trigger ownership restructuring. Power of attorney often signals elder care needs and potential liquidation (3-12 months). Weights: signal=7, distress=6, revenue=7.

```sql control_life_volume
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where subcategory = 'Life and Legal Triggers'
    group by 1
),
with_ma as (
    select record_month, cnt,
        round(avg(cnt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, cnt as value from with_ma
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
    series_order=["Monthly", "6-Mo MA"]
    y_fmt="num0"
    title="Life and Legal Triggers -- Volume Control Chart"
    chart_options={
        series_colors = {
            "Monthly" = "#006BA4"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_life_volume_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}

```sql control_life_dollars
with monthly as (
    select record_month, sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where subcategory = 'Life and Legal Triggers'
    group by 1
),
with_ma as (
    select record_month, amt,
        round(avg(amt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, amt as value from with_ma
union all select record_month, '6-Mo MA', ma_6 from with_ma
order by 1, 2
```

```sql control_life_dollars_limits
with monthly as (
    select record_month, sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where subcategory = 'Life and Legal Triggers'
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
    data="control_life_dollars"
    x="record_month"
    y="value"
    series="series"
    series_order=["Monthly", "6-Mo MA"]
    y_fmt="usd0"
    title="Life and Legal Triggers -- Dollar Volume Control Chart"
    chart_options={
        series_colors = {
            "Monthly" = "#D62728"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_life_dollars_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}


### Deal Propensity Events

Rescissions and amendments. Low volume but meaningful -- an amendment to a mortgage or contract means a deal is being restructured and is actively in progress. A rescission means a deal is unwinding, creating either opportunity (listing back on market) or risk (portfolio uncertainty). Weights: signal=6, distress=1, revenue=6.

```sql control_deal_prop_volume
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where subcategory = 'Deal Propensity Events'
    group by 1
),
with_ma as (
    select record_month, cnt,
        round(avg(cnt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, cnt as value from with_ma
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
    series_order=["Monthly", "6-Mo MA"]
    y_fmt="num0"
    title="Deal Propensity Events -- Volume Control Chart"
    chart_options={
        series_colors = {
            "Monthly" = "#006BA4"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_deal_prop_volume_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}

---

## Signal Family: Transaction Feasibility & Close Intelligence

**The question this answers:** "Will this deal close, and what will block it?"

Title companies, escrow, lenders, and attorneys lose time and money on deals that fall apart due to undetected blockers. This family predicts close readiness and identifies blockers early. The 29 instrument types span three subcategories: title and ownership complexity (quitclaim deeds, affidavits, certificates, corrections), liens and encumbrances (lis pendens, tax rolls), and deal complexity events (assignments, addenda, stipulations).

### Category-Level Overview

```sql control_feasibility_volume
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where category = 'Transaction Feasibility & Close Intelligence'
    group by 1
),
with_ma as (
    select record_month, cnt,
        round(avg(cnt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, cnt as value from with_ma
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
    series_order=["Monthly", "6-Mo MA"]
    y_fmt="num0"
    title="Transaction Feasibility & Close -- Volume Control Chart"
    subtitle="Spikes here mean deals are getting harder to close across the market"
    chart_options={
        series_colors = {
            "Monthly" = "#006BA4"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_feasibility_volume_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}

```sql control_feasibility_dollars
with monthly as (
    select record_month, sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where category = 'Transaction Feasibility & Close Intelligence'
    group by 1
),
with_ma as (
    select record_month, amt,
        round(avg(amt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, amt as value from with_ma
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
    series_order=["Monthly", "6-Mo MA"]
    y_fmt="usd0"
    title="Transaction Feasibility & Close -- Dollar Volume Control Chart"
    chart_options={
        series_colors = {
            "Monthly" = "#D62728"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_feasibility_dollars_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}


### Title and Ownership Complexity

Quitclaim deeds, affidavits (identity, marital status, transferee, contract), certificates (title, estoppel), corrections, amended deeds, memoranda, and horizontal property regime documents. These complicate ownership, title clarity, or property regime structure. A spike here means the market's deal pipeline is getting more complex. Weights: signal=2, distress=2, revenue=2.

```sql control_title_volume
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where subcategory = 'Title and Ownership Complexity'
    group by 1
),
with_ma as (
    select record_month, cnt,
        round(avg(cnt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, cnt as value from with_ma
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
    series_order=["Monthly", "6-Mo MA"]
    y_fmt="num0"
    title="Title and Ownership Complexity -- Volume Control Chart"
    chart_options={
        series_colors = {
            "Monthly" = "#006BA4"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_title_volume_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}

```sql control_title_dollars
with monthly as (
    select record_month, sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where subcategory = 'Title and Ownership Complexity'
    group by 1
),
with_ma as (
    select record_month, amt,
        round(avg(amt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, amt as value from with_ma
union all select record_month, '6-Mo MA', ma_6 from with_ma
order by 1, 2
```

```sql control_title_dollars_limits
with monthly as (
    select record_month, sum(current_month_amount) as amt
    from gold_signal_velocity_trends
    where subcategory = 'Title and Ownership Complexity'
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
    data="control_title_dollars"
    x="record_month"
    y="value"
    series="series"
    series_order=["Monthly", "6-Mo MA"]
    y_fmt="usd0"
    title="Title and Ownership Complexity -- Dollar Volume Control Chart"
    chart_options={
        series_colors = {
            "Monthly" = "#D62728"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_title_dollars_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}


### Deal Complexity Events

Assignments (lease, rents, amendment), addenda, and stipulations. These restructure deal terms or introduce complexity. An assignment of rents signals an income property changing hands. A stipulation signals negotiated terms in litigation affecting property. Weights: signal=4, distress=1, revenue=4.

```sql control_deal_complex_volume
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where subcategory = 'Deal Complexity Events'
    group by 1
),
with_ma as (
    select record_month, cnt,
        round(avg(cnt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, cnt as value from with_ma
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
    series_order=["Monthly", "6-Mo MA"]
    y_fmt="num0"
    title="Deal Complexity Events -- Volume Control Chart"
    chart_options={
        series_colors = {
            "Monthly" = "#006BA4"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_deal_complex_volume_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}

---

## Signal Family: Market Dynamics Intelligence

**The question this answers:** "Where is the market tightening or loosening?"

Zone changes, land use commission actions, permits, easements, and declarations of taking. These are supply pipeline signals -- early indicators of new inventory entering the market. A zone change today means construction permits in 6-12 months and new housing supply in 18-36 months. An easement makes a property more developable. Low volume individually, but when aggregated by geography over time they reveal where the market is expanding.

```sql control_market_volume
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where category = 'Market Dynamics Intelligence'
    group by 1
),
with_ma as (
    select record_month, cnt,
        round(avg(cnt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, cnt as value from with_ma
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
    series_order=["Monthly", "6-Mo MA"]
    y_fmt="num0"
    title="Market Dynamics Intelligence -- Volume Control Chart"
    subtitle="Supply pipeline signals. Breakout = genuine shift in development activity."
    chart_options={
        series_colors = {
            "Monthly" = "#006BA4"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_market_volume_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}

---

## Signal Family: Regulatory, Risk & Claims Intelligence

**The question this answers:** "Where are compliance and loss risks concentrated?"

Waivers, notices of taking, designations, declarations, covenants, and assessments. Low volume but high importance. Assessments create additional financial burden, increasing delinquency risk. Covenants and declarations impose usage restrictions and title complexity. Notice of taking signals eminent domain -- forced sale and displacement. These instruments have low direct revenue value (weights: signal=3, distress=3, revenue=1) but are critical for portfolio risk monitoring and compliance.

```sql control_regulatory_volume
with monthly as (
    select record_month, sum(current_month_count) as cnt
    from gold_signal_velocity_trends
    where category = 'Regulatory, Risk & Claims Intelligence'
    group by 1
),
with_ma as (
    select record_month, cnt,
        round(avg(cnt) over (order by record_month rows between 5 preceding and current row), 0) as ma_6
    from monthly
)
select record_month, 'Monthly' as series, cnt as value from with_ma
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
    series_order=["Monthly", "6-Mo MA"]
    y_fmt="num0"
    title="Regulatory, Risk & Claims -- Volume Control Chart"
    subtitle="Low volume, high compliance importance. Breakout = investigate immediately."
    chart_options={
        series_colors = {
            "Monthly" = "#006BA4"
            "6-Mo MA" = "#898989"
        }
    }
%}
    {% reference_line data="control_regulatory_volume_limits" y="ref_val" label="ref_label" color="#595959" line_options={ type = "dashed" width = 1 } /%}
{% /line_chart %}

---

## Annual Trend

Year-over-year volume by category since 2018. Use for strategic planning, compliance reporting, and multi-year trend analysis.

{% table
    data="gold_yearly_instrument_summary"
    where="category != 'Metadata'"
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

## Appendix A: Signal Classification Reference

RealFACTS classifies every recorded instrument from the Department of Land Management into one of five signal families plus a Metadata category, using the `dim.signal_category` dimension table. Classification is driven by the instrument's type and sub-type fields joined to the dimension table in the bronze layer. Coverage: 165 instrument type/sub-type combinations classified, covering 98.8% of all 993K instruments.

### Categories and What They Mean for a Lending Partner

| Category | Types | What It Captures | Typical Weights (S/D/R) |
|---|---|---|---|
| **Capital & Leverage Intelligence** | 17 | Mortgages, modifications, subordinations, bonds, promissory notes, assumptions | 10 / 2 / 10 |
| **Transaction Propensity Intelligence** | 93 | Deeds, leases, contracts, sales, releases, liens, judgments, probate, forced sales | Varies by subcategory |
| **Transaction Feasibility & Close Intelligence** | 29 | Quitclaim deeds, affidavits, title certificates, assignments, lis pendens, corrections | 2-8 / 1-8 / 2-4 |
| **Market Dynamics Intelligence** | 6 | Zone changes, permits, easements, declarations of taking, bills of sale | 3-8 / 1 / 3-8 |
| **Regulatory, Risk & Claims Intelligence** | 7 | Waivers, notices of taking, designations, declarations, covenants, assessments | 3 / 3 / 1 |
| **Metadata** | 13 | Administrative filings, bylaws, articles, minutes, certificates of title | 1 / 1 / 1 |

### Transaction Propensity Subcategories

| Subcategory | Types | What It Captures | Weights (S/D/R) |
|---|---|---|---|
| **Transaction Timing Triggers** | 23 | Deeds, conveyances, leases, contracts, options, maps | 8 / 1 / 8 |
| **Equity Unlock Signals** | 12 | Mortgage releases, satisfactions, discharges, judgment cancellations | 9 / 1 / 9 |
| **Forced Sale Signals** | 10 | Notices of default/foreclosure, tax deeds, trustee deeds, power of sale | 10 / 10 / 4 |
| **Owner Pressure Signals** | 17 | Liens, judgments, lis pendens, tax lien notices, mechanics liens, writs | 9 / 9 / 3 |
| **Blocker Clearance Signals** | 7 | Cancellations, terminations, withdrawals, dismissals | 5 / 3 / 5 |
| **Life and Legal Triggers** | 22 | Death certificates, divorce decrees, probate, powers of attorney, trusts, gift deeds | 7 / 6 / 7 |
| **Deal Propensity Events** | 2 | Rescissions, amendments | 6 / 1 / 6 |

### Signal Weights

Each instrument type carries three weights on a 1-10 scale:

- **Signal weight (S):** How important is this instrument type as a general indicator of property activity? A foreclosure deed (10) matters more than a letter (1).
- **Distress weight (D):** How strongly does this instrument indicate financial or legal pressure? A writ of execution (9) is high distress; a warranty deed (1) is not.
- **Revenue weight (R):** How directly does this instrument connect to a lending opportunity? A mortgage modification (10) signals an active borrower; a covenant (1) does not.

These three axes feed the weighted scoring engine in the silver and gold layers, enabling work queues to be tuned for different banking objectives: a distress counseling queue weights distress heavily; a refinance capture queue weights revenue heavily.

---

## Appendix B: Control Chart Method

Every control chart on this page uses XmR (Individuals and Moving Range) analysis. Here is how it works in plain terms.

Imagine watching a traffic intersection. Some months 100 cars go through, some months 140, some months 90. That variation is normal -- weather, holidays, random chance. You would not rebuild the road every time the number wiggled.

XmR answers one question: **is this month's number actually different, or is it just normal variation?**

It works by measuring how much the count typically jumps from one month to the next. That jump is the Moving Range -- the absolute difference between consecutive months. If most months jump by about 15-20, that tells you the process has a certain amount of natural wobble. The "X" (Individuals) is just each monthly value plotted over time.

From the average jump size, the method calculates how wide the normal wobble band is. The average moving range is divided by 1.128 (a fixed conversion factor called the d2 constant) to estimate one standard deviation. Control limits are drawn at three standard deviations above and below the mean:

- **Center line (Mean):** Average of all monthly values
- **UCL (Upper Control Limit):** Mean + 3 standard deviations
- **LCL (Lower Control Limit):** Mean - 3 standard deviations (floored at zero)

Anything inside the band is expected variation -- noise. Anything outside has less than a 0.27% chance of occurring under stable conditions. That is a real change worth investigating.

The 3-year lookback window provides 36 data points per subcategory, which is sufficient for stable limit estimation. If Guam's recording patterns show overdispersion (common in large-count data), a Laney u' adjustment can be applied to widen the limits and reduce false signals.

### Moving Averages

The 6-month moving average overlaid on each chart smooths out monthly noise to reveal directional trends. It approximates a 26-week window and shows the medium-term structural trend. When the monthly value diverges sharply from the 6-month average, that divergence is itself a signal -- even if the monthly value stays within control limits.