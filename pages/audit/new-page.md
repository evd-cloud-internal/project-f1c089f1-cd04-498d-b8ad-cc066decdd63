---
name: New Page
assetId: ee60845c-f244-4eb4-8363-84387129309b
type: page
---

---
title: Commercial Lending
---

# Commercial Lending

Individuals holding multiple active mortgages across different properties are portfolio investors operating at a scale that exceeds residential lending products. They need commercial lending relationships: portfolio loans, lines of credit, property management accounts.

---

## Portfolio Summary

{% row %}
{% big_value data="gold_commercial_lending_dashboard" value="sum(party_count)" title="Portfolio Investors" fmt="num0" /%}
{% big_value data="gold_commercial_lending_dashboard" value="sum(total_active_mortgages)" title="Active Mortgages" fmt="num0" /%}
{% big_value data="gold_commercial_lending_dashboard" value="sum(total_mortgage_exposure)" title="Total Exposure" fmt="usd0" /%}
{% big_value data="gold_commercial_lending_dashboard" value="sum(parties_with_contact)" title="With Contact Info" fmt="num0" /%}
{% /row %}

---

## Portfolio Tier Distribution

{% horizontal_bar_chart
    data="gold_commercial_lending_dashboard"
    x="portfolio_tier"
    y="sum(party_count)"
    title="Investors by Portfolio Tier"
/%}

{% combo_chart
    data="gold_commercial_lending_dashboard"
    x="portfolio_tier"
    y_fmt="num0"
    y2_fmt="usd0"
    title="Investors and Exposure by Tier"
%}
    {% bar y="sum(party_count)" /%}
    {% line y="sum(total_mortgage_exposure)" axis="y2" /%}
{% /combo_chart %}

{% table data="gold_commercial_lending_dashboard" %}
    {% dimension value="portfolio_tier" /%}
    {% measure value="sum(party_count)" fmt="num0" /%}
    {% measure value="sum(total_active_mortgages)" fmt="num0" /%}
    {% measure value="sum(total_mortgage_exposure)" fmt="usd0" viz="bar" /%}
    {% measure value="max(avg_years_as_borrower)" fmt="num1" /%}
{% /table %}

---

## Work Queue: Commercial Lending Leads

{% table data="report_commercial_lending_leads" limit=50 %}
    {% dimension value="party_name" /%}
    {% measure value="max(active_mortgages)" fmt="num0" /%}
    {% measure value="max(total_exposure)" fmt="usd0" /%}
    {% measure value="max(property_count)" fmt="num0" /%}
    {% measure value="max(years_as_borrower)" fmt="num1" /%}
{% /table %}