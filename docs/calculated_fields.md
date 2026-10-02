# Calculated fields

The fields behind `1. Final`, in the order they build on each other. One line of "why" each; the
formulas are exactly as in the workbook.

Source grain: **environment × casino × provider × day**. The table also carries casino-level rows,
marked by `Grouping Set Level`, so every FIXED expression guards on
`[Grouping Set Level] = "casino-provider"` to avoid counting them twice.

---

## 1. Windows

```
Report Day      = { MAX([Date Only]) }
Date Start _ 7  = DATE(DATEADD('day',  -6, [Report Day]))    -- this week starts here
Date Start _ 50 = DATE(DATEADD('day', -49, [Report Day]))    -- baseline starts here
```

Why `MAX(date)` and not `TODAY()`: data lands hours after midnight. Anchoring to the clock would
show an empty last day as a wave of false alerts every morning.

- **Baseline** = `Date Start _ 50` to the day before `Date Start _ 7` (43 days)
- **This week** = `Date Start _ 7` to `Report Day` (7 days)

## 2. Flag → date

```
Date _ Have Traffic = IF [Traffic Flag] = 'Have Traffic' THEN [Date Only] END
```

A nullable date lets `COUNTD` count "days with traffic" with no conditional aggregation. Never
had traffic → `NULL`, not `0`.

## 3. Zero-traffic days this week

```
Casino Provider ZERO traffic days =                       -- per provider → cell color
  7 - { FIXED [Environment code], [Casino ID], [Game Provider] :
        COUNTD( IF [Date Only] >= [Date Start _ 7] THEN [Date _ Have Traffic] END ) }

Zero Traffic Days =                                       -- whole casino → "No traffic entire casino"
  7 - { FIXED [Environment code], [Casino ID] :
        COUNTD( IF [Date Only] >= [Date Start _ 7] THEN [Date _ Have Traffic] END ) }
```

Both unchanged from the original report: the redesign changed what is *shown*, not what counts as silence.

## 4. Baseline rate

```
Provider Avg Daily Wager =
  { FIXED [Environment code], [Casino ID], [Game Provider] :
      SUM( IF [Grouping Set Level] = "casino-provider"
           AND [Date Only] >= [Date Start _ 50] AND [Date Only] < [Date Start _ 7]
           THEN [Wager Eur] END ) }
  /
  { FIXED [Environment code], [Casino ID], [Game Provider] :
      COUNTD( IF [Grouping Set Level] = "casino-provider"
              AND [Date Only] >= [Date Start _ 50] AND [Date Only] < [Date Start _ 7]
              THEN [Date _ Have Traffic] END ) }
```

Baseline wager ÷ baseline days **with traffic**. Silent days do not dilute it, so a stopped casino
keeps its real rate and a new casino is measured from day one. No baseline traffic → divide by zero
→ `NULL` → never enters the gates below. The tooltip shows how many days the average rests on.

## 5. Potential

```
Provider Gap = [Provider Avg Daily Wager] * [Casino Provider ZERO traffic days]
```

"At its normal rate, what did the silent days cost?" Bounded: at most 7 days of the provider's own
normal.

## 6. One rule, four places

The rule: **zero days ≥ 1 AND gap ≥ `Min potential wager per provider`**. It appears in four fields
because Tableau needs it as a filter, a number, a color and a total — but it is the same rule, so
the row total always equals the sum of the colored cells.

```
-- entry filter (dimension filter = True)
Has no-traffic provider this week =
  { FIXED [Environment code], [Casino ID] :
      MAX( IF [Grouping Set Level] = "casino-provider"
           AND [Casino Provider ZERO traffic days] >= 1
           AND [Provider Gap] >= [Min potential wager per provider]
           THEN 1 ELSE 0 END ) } = 1

-- cell number
Missing this week =
  IF [Casino Provider ZERO traffic days] >= 1
     AND [Provider Gap] >= [Min potential wager per provider]
  THEN [Provider Gap] END

-- cell color ('' is mapped to white)
No traffic severity =
  IF IFNULL([Provider Gap], 0) < [Min potential wager per provider] THEN ''
  ELSEIF [Casino Provider ZERO traffic days] = 7 THEN '7 days no traffic'
  ELSEIF [Casino Provider ZERO traffic days] = 6 THEN '6 days no traffic'
  ...
  ELSEIF [Casino Provider ZERO traffic days] = 1 THEN '1 day no traffic'
  ELSE '' END

-- row total (Rows shelf, sort key)
Potential Wager =
  { FIXED [Environment code], [Casino ID] :
      SUM( { FIXED [Environment code], [Casino ID], [Game Provider] :
               MAX( IF [Grouping Set Level] = "casino-provider"
                    AND [Casino Provider ZERO traffic days] >= 1
                    AND [Provider Gap] >= [Min potential wager per provider]
                    THEN [Provider Gap] END ) } ) }
```

The nested FIXED in `Potential Wager`: inner one collapses each provider's day rows to one value,
outer one adds the providers up.

## 7. Parameter

`Min potential wager per provider` — integer, default **1,000**. Used in the four fields above and
in the header text, so one change updates the rule and its explanation together. Why 1,000:
`design-decisions.md` §7.

## 8. 50-day list

```
Casino Last Traffic Day  = { FIXED [Environment code], [Casino ID] : MAX([Date _ Have Traffic]) }
Casino no traffic filter = ISNULL([Casino Last Traffic Day])
Casino Go Live Date      = { FIXED [Environment code], [Casino ID] : MIN([Go Live Date]) }
```

On the list = no traffic day at all in the window. Go-live date tells "long dead" from "not started".

---

## Things that break silently

- **Never put the 7-day filter in context.** Context filters run before FIXED LODs; a 7-day context
  filter collapses every baseline to this week. No error, wrong numbers.
- **Row-level security *is* a context filter**, on purpose: each manager's baselines are computed
  within their own portfolio. Stubbed to `TRUE` in this public copy.
- **`Grouping Set Level` is a normal filter**, so FIXED LODs ignore it — that is why each LOD
  carries its own `= "casino-provider"` guard.
- **Aggregating a FIXED field with `SUM`** on a shelf multiplies it by the row count. Use `MIN`,
  `MAX` or `ATTR`.
- **The color map lives in the datasource**, not the worksheet. A worksheet-level palette is
  ignored.
