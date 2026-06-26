# Make platform spend filter by Source (bing → ~£729)

The spend spine currently stops at **channel_group**, which lumps Bing+Google+Facebook
together. Add **source** as a dimension of the spine scaffold and to the two joins.
The spine stays complete (no-lead days still covered, totals unchanged); it's just
split by source. No de-dup needed — the spine hub prevents fan-out.

Do this in Tableau Desktop on the live connection, then **Refresh Extract**.

---

## 1. Spine — `Custom SQL Query` (replace the whole query with this)

```sql
select s.calendar_date, c.country_iso, ch.channel_group, bu.business_unit, src.source
from (
    select dateadd(day, seq, date '2024-01-01')::date as calendar_date
    from (
        select row_number() over (order by campaign_id) - 1 as seq
        from digital_marketing_agg.platform_spend_5_campaign
    ) g
    where dateadd(day, seq, date '2024-01-01') <= date '2027-12-31'
) s
cross join (
    select 'GB' as country_iso union all select 'IE' union all select 'BE' union all
    select 'DE' union all select 'ES' union all select 'IT' union all select 'NL' union all
    select 'PT' union all select 'FR' union all select 'AU' union all select 'NZ' union all
    select 'SG' union all select 'MY' union all select 'US'
) c
cross join (
    select 'iCompario' as channel_group union all select 'Radius.com' union all
    select 'DCI Card' union all select 'Radius Fuel Solutions' union all
    select 'Radius Telematics' union all select 'Social Media' union all
    select 'Self-Source' union all select 'Divisional Referral' union all
    select 'Refer-a-friend' union all select 'Inbound (Call/Email/Website)' union all
    select 'Other' union all
    select 'AdStrategy' union all select 'Addmodum' union all
    select 'Bizzbooster' union all select 'Bobex' union all
    select 'Bouwunie' union all select 'BP' union all
    select 'CIB' union all select 'Companeo' union all
    select 'eRocket' union all select 'Flotten Vergleich' union all
    select 'Fluxa' union all select 'Ik-rij-elektrisch' union all
    select 'Laadpalenwijzer' union all select 'Laadpascom' union all
    select 'Laadpastop10' union all select 'Leadboost' union all
    select 'Lead The Way' union all select 'MVF' union all
    select 'Onssen' union all select 'Private Partners' union all
    select 'Tankkaart-aanvragen' union all select 'Tankpas-aanvragen' union all
    select 'Top Partner' union all select 'TradingTwins' union all
    select 'Verkeersonderneming' union all select 'Wijsvergelijken' union all
    select 'ZionMedia' union all select 'Other 3rd Party'
) ch
cross join (
    select 'Fuel' as business_unit union all select 'Telematics' union all select 'Other'
) bu
cross join (
    select 'Bing' as source union all select 'Google' union all select 'Facebook'
) src
```

(Only the last `cross join (...) src` and `, src.source` in the SELECT are new.)

---

## 2. Bridge — `Custom SQL Query2` (add one column to the SELECT)

Add this near `l.datatracker_source,` — it normalises the lead's source to the same
casing the spend table uses (lead side is `bing`, spend side is `Bing`):

```sql
    CASE LOWER(l.datatracker_track_source)
        WHEN 'bing'     THEN 'Bing'
        WHEN 'google'   THEN 'Google'
        WHEN 'facebook' THEN 'Facebook'
        ELSE l.datatracker_track_source
    END AS spend_source,
```

---

## 3. Relationships (the join "noodles" in the data-source pane)

**spine (`Custom SQL Query`) ↔ platform_spend (`Custom SQL Query3`)**
Currently: `calendar_date = date`, `country_iso = country_iso`, `channel_group = channel_group`, `business_unit = business_unit`.
**Add one more pair:**  `source  =  source`

**bridge (`Custom SQL Query2`) ↔ spine (`Custom SQL Query`)**
Currently: `created_day = calendar_date`, `country_iso = country_iso`, `channel_group = channel_group`, `business_unit = business_unit`.
**Add one more pair:**  `spend_source  =  source`

Both are exact matches, so you can pick them straight from the dropdowns — no calc needed.

---

## 4. Refresh & verify
1. Data → Extract → **Refresh** (this materialises the new `source` column).
2. Weekly summary, no Source filter: totals should match today's.
3. Set Source = **bing**: spend should drop to ~£729 (week 24), leads unchanged.

If a channel subtotal moves slightly, that's spend that genuinely had no lead in that
exact date/country/channel — expected, and it stays in the country/week totals.
