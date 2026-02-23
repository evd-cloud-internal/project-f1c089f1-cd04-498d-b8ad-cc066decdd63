---
name: 09 - Refi & Retention
assetId: ee60845c-f244-4eb4-8363-84387129309b
type: page
---

# Refi & Retention Pipeline

You're losing customers right now. Every mortgage release and satisfaction in this pipeline means a borrower either refinanced with a competitor or paid off completely. The first section shows what you're missing. The second shows what you can win. The third is your call sheet.

For the rate-aware version of this pipeline -- which mortgages actually pencil out for refi given today's rates -- see the **Lending POC Work Queues** page.

---

## Reference: How to Read This Page

### Priority Tier (Retention Alerts)

How recently the equity unlock signal was recorded. Time is critical -- if they refinanced with a competitor, the window to win them back closes fast.

| Tier | Window | Action |
|---|---|---|
| **URGENT - This Week** | 0-7 days | Immediate outreach. Highest win-back probability. |
| **HIGH - Last 2 Weeks** | 8-14 days | Same-week call. Still warm. |
| **MEDIUM - Last Month** | 15-30 days | Outreach before they settle into new lender. |
| **MONITOR** | 31-90 days | Cooling. HELOC pitch may still apply if they paid off. |

### Refi Timing Window (Sweet Spot)

A conventional sales prospecting heuristic for loan officer outreach -- not an eligibility gate. Borrowers can refinance at any age, and government streamline programs (VA IRRRL, FHA, USDA) qualify as early as 6-7 months. The 24-60 month window is where conventional refi outreach is traditionally concentrated because seasoning requirements and closing cost break-even tend to align.

| Window | Age | Typical Pitch |
|---|---|---|
| **PRIME - 2-3 Years** | 24-36 months | Strongest conventional outreach window. Rate reduction if environment is favorable. |
| **GOOD - 3-4 Years** | 37-48 months | Still strong. Borrower may also want cash-out as equity builds. |
| **LATE WINDOW - 4-5 Years** | 49-60 months | End of traditional sweet spot. Higher equity = HELOC or rate+term pitch. |

**This page is timing-only.** It does not consider the rate environment. A mortgage in the sweet spot that was originated below today's rate is a retention target, not a refi candidate. The Rate Gap Portfolio and Lending POC Work Queues pages add that financial layer.

### Value Tier

Dollar-based prioritization of the mortgage principal (thresholds pending Ryan confirmation).

---

## Equity Unlock Signals

Equity unlock signals from the last 90 days -- mortgage releases, satisfactions, reconveyances. Each one is a borrower who just left or a HELOC opportunity sitting idle.

{% row %}
    {% big_value
        data="report_lender_retention_alerts"
        value="count(*)"
        title="Total Retention Alerts"
        fmt="num0"
    /%}
    {% big_value
        data="report_lender_retention_alerts"
        value="sum(amount)"
        title="Total Dollar Volume"
        fmt="usd0"
    /%}
    {% big_value
        data="report_lender_retention_alerts"
        value="count(*)"
        where="priority_tier = 'URGENT - This Week'"
        title="Urgent (This Week)"
        fmt="num0"
    /%}
    {% big_value
        data="report_lender_retention_alerts"
        value="count(*)"
        where="priority_tier = 'HIGH - Last 2 Weeks'"
        title="High (Last 2 Weeks)"
        fmt="num0"
    /%}
{% /row %}

{% bar_chart
    data="report_lender_retention_alerts"
    x="priority_tier"
    y="count(*)"
    y_fmt="num0"
    title="Retention Alerts by Priority"
    subtitle="90-day window -- how much time do you have?"
    chart_options={
        color_palette = ["#006BA4", "#FF800E", "#5F9ED1", "#C85200", "#595959"]
    }
/%}

### Alert Detail

```sql retention_detail
select
    instrument_number as "Instrument",
    date_recorded as "Date Recorded",
    amount as "Amount",
    equity_unlock_type as "Unlock Type",
    asset_type_released as "Asset Released",
    days_since_recorded as "Days Ago",
    priority_tier as "Priority",
    name_full as "Contact Name",
    phone_number as "Phone",
    email as "Email",
    address_city as "City"
from report_lender_retention_alerts
order by date_recorded desc
```

{% table data="retention_detail" /%}

---

## Here's Your Growth Pipeline

Active mortgages in the 24-60 month prospecting window. Three timing sub-windows tell your loan officers where outreach is traditionally concentrated. For which of these actually pencil out at today's rates, see the Rate Gap Portfolio page.

{% row %}
    {% big_value
        data="report_refi_sweet_spot"
        value="count(*)"
        title="Sweet Spot Mortgages"
        fmt="num0"
    /%}
    {% big_value
        data="report_refi_sweet_spot"
        value="sum(principal)"
        title="Total Principal"
        fmt="usd0"
    /%}
    {% big_value
        data="report_refi_sweet_spot"
        value="count(*)"
        where="refi_timing = 'PRIME - 2-3 Years'"
        title="Prime Window (2-3yr)"
        fmt="num0"
    /%}
    {% big_value
        data="report_refi_sweet_spot"
        value="count(*)"
        where="refi_timing = 'GOOD - 3-4 Years'"
        title="Good Window (3-4yr)"
        fmt="num0"
    /%}
{% /row %}

{% bar_chart
    data="report_refi_sweet_spot"
    x="refi_timing"
    y="count(*)"
    series="mortgage_type_group"
    y_fmt="num0"
    title="Sweet Spot Mortgages by Timing Window & Product"
    subtitle="Where is the volume by product type?"
    chart_options={
        color_palette = ["#006BA4", "#FF800E", "#5F9ED1", "#C85200", "#595959", "#FFBC79", "#A2C8EC", "#898989"]
    }
/%}

### Sweet Spot Detail

```sql sweet_spot_detail
select
    instrument_number as "Instrument",
    mortgage_type_group as "Product Group",
    mortgage_trusted_type as "Product Type",
    date_executed as "Originated",
    principal as "Principal",
    months_since_executed as "Months",
    vintage_year as "Vintage",
    refi_timing as "Timing Window",
    value_tier as "Value Tier",
    name_full as "Contact Name",
    phone_number as "Phone",
    email as "Email",
    address_city as "City"
from report_refi_sweet_spot
order by principal desc nulls last
```

{% table data="sweet_spot_detail" /%}

---

## Make the Call

The lean outreach list. Same pipeline, stripped to what a loan officer needs to pick up the phone.

```sql outreach_list
select
    instrument_number as "Instrument",
    mortgage_type_group as "Product",
    date_executed as "Originated",
    principal as "Principal",
    months_since_executed as "Months",
    value_tier as "Value Tier",
    name_full as "Contact",
    phone_number as "Phone",
    email as "Email",
    address_city as "City"
from report_refi_outreach_list
order by principal desc nulls last
```

{% table data="outreach_list" /%}